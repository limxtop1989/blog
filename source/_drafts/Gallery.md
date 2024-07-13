---
title: Gallery
date: 2024-01-19 15:57:06
tags:
---


# Data
* BUCKET_DISPLAY_NAME is a string like "Camera" which is the directory name of where an image or video is in.
* BUCKET_ID is a hash of the path name of that directory (see computeBucketValues() in MediaProvider for details). 
* MEDIA_TYPE is video, image, audio, etc.
* The "albums" are not explicitly recorded in the database, but each image or video has the two columns (BUCKET_ID, MEDIA_TYPE). We define an "album" to be the collection of images/videos which have the same value for the two columns.

* MediaItem represents an image or a video item.
    * LocalVideo represents a video in the local storage.
    * LocalImage represents an image in the local storage.
* MediaSet is a directory-like data structure. It contains MediaItems and sub-MediaSets.
    * LocalAlbum lists all media items in one bucket on local storage. The media items need to be all images or all videos, but not both.
    * LocalAlbumSet lists all image or video albums in the local storage. The path should be "/local/image", "local/video" or "/local/all"
    * FilterDeleteSet filters a base MediaSet to remove some deletion items (we expect the number to be small)
    * ComboAlbum combines multiple media sets into one. It lists all media items from the input albums. This only handles SubMediaSets, not MediaItems. (That's all we need now)
    * ComboAlbumSet combines multiple media sets into one. It lists all sub media sets from the input album sets. This only handles SubMediaSets, not MediaItems.
    * LocalMergeAlbum merges items from two or more MediaSets. It uses a Comparator to determine the order of items. The items are assumed to be sorted in the input media sets (with the same order that the Comparator uses). This only handles MediaItems, not SubMediaSets.
    * SnailAlbum is a simple MediaSet which contains only one MediaItem -- a SnailItem.
    * FilterTypeSet filters a base MediaSet according to a matching media type.
    * ClusterAlbum
    * ClusterAlbumSet


# DataManager manages all media sets and media items in the system.
Each MediaSet and MediaItem has a unique 64 bits id. The most significant
32 bits represents its parent, and the least significant 32 bits represents
the self id. For MediaSet the self id is is globally unique, but for
MediaItem it's unique only relative to its parent.
# TODO
1. What is picasa?

## onCreate
com.android.gallery3d.app.AlbumSetPage#onCreate
    com.android.gallery3d.app.AlbumSetPage#initializeData
```java
// com.android.gallery3d.app.GalleryActivity#startDefaultPage
        Bundle data = new Bundle();
        data.putString(AlbumSetPage.KEY_MEDIA_PATH,
                getDataManager().getTopSetPath(DataManager.INCLUDE_ALL));// "/combo/{/local/all,/picasa/all}"
        getStateManager().startState(AlbumSetPage.class, data);

// com.android.gallery3d.data.ComboSource#createMediaObject
// path: /combo/{/local/all,/picasa/all}
    @Override
    public MediaObject createMediaObject(Path path) {
        String[] segments = path.split();
        if (segments.length < 2) {
            throw new RuntimeException("bad path: " + path);
        }

        DataManager dataManager = mApplication.getDataManager();
        switch (mMatcher.match(path)) {
            case COMBO_ALBUMSET:
                return new ComboAlbumSet(path, mApplication,
                        dataManager.getMediaSetsFromString(segments[1]));

            case COMBO_ALBUM:
                return new ComboAlbum(path,
                        dataManager.getMediaSetsFromString(segments[2]), segments[1]);
        }
        return null;
    }

addSource(new ComboSource(mApplication)); -> new ComboAlbumSet(path, mApplication,
                        dataManager.getMediaSetsFromString(segments[1]));
        addSource(new LocalSource(mApplication));  -> new LocalAlbumSet(path, mApplication);
        addSource(new PicasaSource(mApplication)); -> new EmptyAlbumSet(path, MediaObject.nextVersionNumber());
```

## onResume
com.android.gallery3d.app.AbstractGalleryActivity#onResume
    com.android.gallery3d.app.StateManager#resume
    com.android.gallery3d.app.ActivityState#resume
    com.android.gallery3d.app.AlbumSetPage#onResume
    com.android.gallery3d.app.AlbumSetDataLoader#resume


```java
    private static class UpdateInfo {
        public long version;
        public int reloadStart;
        public int reloadCount;

        public int size; // Record size in the table.
        public ArrayList<MediaItem> items; // The items between reloadStart and reloadStart + reloadCount return by query.
    }
```


```java
com.android.gallery3d.ui.AlbumSlidingWindow#requestNonactiveImages

    // We would like to request non active slots in the following order:
    // Order:    8 6 4 2                   1 3 5 7
    //         |---------|---------------|---------|
    //                   |<-  active  ->|
    //         |<-------- cached range ----------->|
    private void requestNonactiveImages() {
        int range = Math.max(
                (mContentEnd - mActiveEnd), (mActiveStart - mContentStart));
        for (int i = 0 ;i < range; ++i) {
            requestSlotImage(mActiveEnd + i);
            requestSlotImage(mActiveStart - 1 - i);
        }
    }

com.android.gallery3d.ui.AlbumSlidingWindow#setActiveWindow
    public void setActiveWindow(int start, int end) {
        if (!(start <= end && end - start <= mData.length && end <= mSize)) {
            Utils.fail("%s, %s, %s, %s", start, end, mData.length, mSize);
        }
        AlbumEntry data[] = mData;

        mActiveStart = start;
        mActiveEnd = end;

        int contentStart = Utils.clamp((start + end) / 2 - data.length / 2,
                0, Math.max(0, mSize - data.length));
        int contentEnd = Math.min(contentStart + data.length, mSize);
        setContentWindow(contentStart, contentEnd);
        updateTextureUploadQueue();
        if (mIsActive) updateAllImageRequests();
    }
```