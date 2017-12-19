# kubelet Image 垃圾回收

```
作者 张杰  j.zhang8@haihangyun.com
```

kubelet 会定期进行垃圾回收，具体垃圾回收规则
cmd/kubelet/app/server.go
CreateAndInitKubelet() 方法里在构建完kubelet 对象后会调用 k.StartGarbageCollection() 方法
这个方法主要用来回收不用的image 和 容器， 代码如下
``` golang
// StartGarbageCollection starts garbage collection threads.
func (kl *Kubelet) StartGarbageCollection() {
	loggedContainerGCFailure := false
	go wait.Until(func() {
		if err := kl.containerGC.GarbageCollect(); err != nil {
			glog.Errorf("Container garbage collection failed: %v", err)
			kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.ContainerGCFailed, err.Error())
			loggedContainerGCFailure = true
		} else {
			var vLevel glog.Level = 4
			if loggedContainerGCFailure {
				vLevel = 1
				loggedContainerGCFailure = false
			}

			glog.V(vLevel).Infof("Container garbage collection succeeded")
		}
	}, ContainerGCPeriod, wait.NeverStop)

	prevImageGCFailed := false
	go wait.Until(func() {
		if err := kl.imageManager.GarbageCollect(); err != nil {
			if prevImageGCFailed {
				glog.Errorf("Image garbage collection failed multiple times in a row: %v", err)
				// Only create an event for repeated failures
				kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.ImageGCFailed, err.Error())
			} else {
				glog.Errorf("Image garbage collection failed once. Stats initialization may not have completed yet: %v", err)
			}
			prevImageGCFailed = true
		} else {
			var vLevel glog.Level = 4
			if prevImageGCFailed {
				vLevel = 1
				prevImageGCFailed = false
			}

			glog.V(vLevel).Infof("Image garbage collection succeeded")
		}
	}, ImageGCPeriod, wait.NeverStop)
}
```
我们着重先看 kl.imageManager.GarbageCollect() 方法， 代码追踪：
pkg/kubelet/images/image_gc_manager.go
```
func (im *realImageGCManager) GarbageCollect() error {
	// Get disk usage on disk holding images.
	fsStats, err := im.statsProvider.ImageFsStats()
	if err != nil {
		return err
	}

	var capacity, available int64
	if fsStats.CapacityBytes != nil {
		capacity = int64(*fsStats.CapacityBytes)
	}
	if fsStats.AvailableBytes != nil {
		available = int64(*fsStats.AvailableBytes)
	}

	if available > capacity {
		glog.Warningf("available %d is larger than capacity %d", available, capacity)
		available = capacity
	}

	// Check valid capacity.
	if capacity == 0 {
		err := goerrors.New("invalid capacity 0 on image filesystem")
		im.recorder.Eventf(im.nodeRef, v1.EventTypeWarning, events.InvalidDiskCapacity, err.Error())
		return err
	}

	// If over the max threshold, free enough to place us at the lower threshold.
	usagePercent := 100 - int(available*100/capacity)
	if usagePercent >= im.policy.HighThresholdPercent {
		amountToFree := capacity*int64(100-im.policy.LowThresholdPercent)/100 - available
		glog.Infof("[imageGCManager]: Disk usage on image filesystem is at %d%% which is over the high threshold (%d%%). Trying to free %d bytes", usagePercent, im.policy.HighThresholdPercent, amountToFree)
		// 主要方法
		freed, err := im.freeSpace(amountToFree, time.Now())
		if err != nil {
			return err
		}

		if freed < amountToFree {
			err := fmt.Errorf("failed to garbage collect required amount of images. Wanted to free %d bytes, but freed %d bytes", amountToFree, freed)
			im.recorder.Eventf(im.nodeRef, v1.EventTypeWarning, events.FreeDiskSpaceFailed, err.Error())
			return err
		}
	}

	return nil
}

```
核心方法： im.freeSpace（）
继续追：
``` golang

func (im *realImageGCManager) freeSpace(bytesToFree int64, freeTime time.Time) (int64, error) {
	err := im.detectImages(freeTime)
	if err != nil {
		return 0, err
	}

	im.imageRecordsLock.Lock()
	defer im.imageRecordsLock.Unlock()

	// Get all images in eviction order.
	images := make([]evictionInfo, 0, len(im.imageRecords))
	for image, record := range im.imageRecords {
		images = append(images, evictionInfo{
			id:          image,
			imageRecord: *record,
		})
	}
	sort.Sort(byLastUsedAndDetected(images))

	// Delete unused images until we've freed up enough space.
	var deletionErrors []error
	spaceFreed := int64(0)
	for _, image := range images {
		glog.V(5).Infof("Evaluating image ID %s for possible garbage collection", image.id)
		// Images that are currently in used were given a newer lastUsed.
		if image.lastUsed.Equal(freeTime) || image.lastUsed.After(freeTime) {
			glog.V(5).Infof("Image ID %s has lastUsed=%v which is >= freeTime=%v, not eligible for garbage collection", image.id, image.lastUsed, freeTime)
			break
		}

		// Avoid garbage collect the image if the image is not old enough.
		// In such a case, the image may have just been pulled down, and will be used by a container right away.

		if freeTime.Sub(image.firstDetected) < im.policy.MinAge {
			glog.V(5).Infof("Image ID %s has age %v which is less than the policy's minAge of %v, not eligible for garbage collection", image.id, freeTime.Sub(image.firstDetected), im.policy.MinAge)
			continue
		}

		// Remove image. Continue despite errors.
		glog.Infof("[imageGCManager]: Removing image %q to free %d bytes", image.id, image.size)
		err := im.runtime.RemoveImage(container.ImageSpec{Image: image.id})
		if err != nil {
			deletionErrors = append(deletionErrors, err)
			continue
		}
		delete(im.imageRecords, image.id)
		spaceFreed += image.size

		if spaceFreed >= bytesToFree {
			break
		}
	}

	if len(deletionErrors) > 0 {
		return spaceFreed, fmt.Errorf("wanted to free %d bytes, but freed %d bytes space with errors in image deletion: %v", bytesToFree, spaceFreed, errors.NewAggregate(deletionErrors))
	}
	return spaceFreed, nil
}
```

这里面会涉及到 imageRecords 变量，这个变量很重要，很多人会疑惑这个变量是在哪里复制个更改的。
here man：
pkg/kubelet/kubelet.go Run() 方法 -> kl.initializeModules（）方法 -> kl.imageManager.Start() 方法

pkg/kubelet/images/image_gc_manager.go  start()方法
``` golang
func (im *realImageGCManager) Start() {
	go wait.Until(func() {
		// Initial detection make detected time "unknown" in the past.
		var ts time.Time
		if im.initialized {
			ts = time.Now()
		}
		// 探测image 主要是更新 imageRecords 参数， 尤其是 lastUsed 参数，代表了最后使用的时间
		err := im.detectImages(ts)
		if err != nil {
			glog.Warningf("[imageGCManager] Failed to monitor images: %v", err)
		} else {
			im.initialized = true
		}
	}, 5*time.Minute, wait.NeverStop)

	// Start a goroutine periodically updates image cache.
	// TODO(random-liu): Merge this with the previous loop.
	go wait.Until(func() {
		images, err := im.runtime.ListImages()
		if err != nil {
			glog.Warningf("[imageGCManager] Failed to update image list: %v", err)
		} else {
			im.imageCache.set(images)
		}
	}, 30*time.Second, wait.NeverStop)

}

```
start 里会调用detectImages 方法，此方法着重对imageRecords 变量进行了赋值
```
func (im *realImageGCManager) detectImages(detectTime time.Time) error {
	images, err := im.runtime.ListImages()
	if err != nil {
		return err
	}
	pods, err := im.runtime.GetPods(true)
	if err != nil {
		return err
	}

	// Make a set of images in use by containers.
	imagesInUse := sets.NewString()
	for _, pod := range pods {
		for _, container := range pod.Containers {
			glog.V(5).Infof("Pod %s/%s, container %s uses image %s(%s)", pod.Namespace, pod.Name, container.Name, container.Image, container.ImageID)
			imagesInUse.Insert(container.ImageID)
		}
	}

	// Add new images and record those being used.
	now := time.Now()
	currentImages := sets.NewString()
	im.imageRecordsLock.Lock()
	defer im.imageRecordsLock.Unlock()
	for _, image := range images {
		glog.V(5).Infof("Adding image ID %s to currentImages", image.ID)
		currentImages.Insert(image.ID)

		// New image, set it as detected now.
		if _, ok := im.imageRecords[image.ID]; !ok {
			glog.V(5).Infof("Image ID %s is new", image.ID)
			im.imageRecords[image.ID] = &imageRecord{
				firstDetected: detectTime,
			}
		}

		// Set last used time to now if the image is being used.
		if isImageUsed(image, imagesInUse) {
			glog.V(5).Infof("Setting Image ID %s lastUsed to %v", image.ID, now)
			im.imageRecords[image.ID].lastUsed = now
		}

		glog.V(5).Infof("Image ID %s has size %d", image.ID, image.Size)
		im.imageRecords[image.ID].size = image.Size
	}

	// Remove old images from our records.
	for image := range im.imageRecords {
		if !currentImages.Has(image) {
			glog.V(5).Infof("Image ID %s is no longer present; removing from imageRecords", image)
			delete(im.imageRecords, image)
		}
	}

	return nil
}

```
此方法最主要的逻辑就是记录image 最后使用的时间。
我们在返回到 

核心方法： im.freeSpace（）

``` golang

func (im *realImageGCManager) freeSpace(bytesToFree int64, freeTime time.Time) (int64, error) {
	err := im.detectImages(freeTime)
	if err != nil {
		return 0, err
	}

	im.imageRecordsLock.Lock()
	defer im.imageRecordsLock.Unlock()

	// Get all images in eviction order.
	images := make([]evictionInfo, 0, len(im.imageRecords))
	for image, record := range im.imageRecords {
		images = append(images, evictionInfo{
			id:          image,
			imageRecord: *record,
		})
	}
	sort.Sort(byLastUsedAndDetected(images))

	// Delete unused images until we've freed up enough space.
	var deletionErrors []error
	spaceFreed := int64(0)
	for _, image := range images {
		glog.V(5).Infof("Evaluating image ID %s for possible garbage collection", image.id)
		// Images that are currently in used were given a newer lastUsed.
		if image.lastUsed.Equal(freeTime) || image.lastUsed.After(freeTime) {
			glog.V(5).Infof("Image ID %s has lastUsed=%v which is >= freeTime=%v, not eligible for garbage collection", image.id, image.lastUsed, freeTime)
			break
		}

		// Avoid garbage collect the image if the image is not old enough.
		// In such a case, the image may have just been pulled down, and will be used by a container right away.

		if freeTime.Sub(image.firstDetected) < im.policy.MinAge {
			glog.V(5).Infof("Image ID %s has age %v which is less than the policy's minAge of %v, not eligible for garbage collection", image.id, freeTime.Sub(image.firstDetected), im.policy.MinAge)
			continue
		}

		// Remove image. Continue despite errors.
		glog.Infof("[imageGCManager]: Removing image %q to free %d bytes", image.id, image.size)
		err := im.runtime.RemoveImage(container.ImageSpec{Image: image.id})
		if err != nil {
			deletionErrors = append(deletionErrors, err)
			continue
		}
		delete(im.imageRecords, image.id)
		spaceFreed += image.size

		if spaceFreed >= bytesToFree {
			break
		}
	}

	if len(deletionErrors) > 0 {
		return spaceFreed, fmt.Errorf("wanted to free %d bytes, but freed %d bytes space with errors in image deletion: %v", bytesToFree, spaceFreed, errors.NewAggregate(deletionErrors))
	}
	return spaceFreed, nil
}
```
在遍历所有images 缓存时候，如果image 的lasteUsed 时间小于当前时间，并且当前时间 减去 firstDetected 时间 大于最小间隔，删掉image
