---
layout: post
title: Picasso Library 가 캐싱하는 법
tags: Picasso, Android
---

Android에서는 이미지 같은 대용량 리소스 파일들을 로컬, 원격에서 받아서 화면에 표시할 경우 리사이징, 캐싱 등을 사용하여 최대한 효율적으로 다뤄야 합니다. 특히 인터넷 상에서 네트워크 연결을 통해 이미지를 받아와 표시할 경우 네트워크 통신 비용 + 이미지 출력 비용이 같이 발생하므로 위에서 언급한 캐싱이 필수적으로 동작해야 합니다. 이번 글에서는 Android에서 사용하는 Image Opensource Library인 Picasso가 어떻게 캐싱을 처리하는지 알아보겠습니다.

Picasso는 square사에서 만든 이미지 관련 라이브러리로 [https://github.com/square/picasso](https://github.com/square/picasso)에 소개가 되어있습니다. 자세한 사용방법은 [https://square.github.io/picasso](https://square.github.io/picasso)에 참고하시면 되겠습니다.
Picasso는 In-memory Cache과 Storage Cache를 사용하여 이미지들을 캐싱하고 있습니다. In-memory Cache구현 방식은 Android의 LruCache<K key, V value> 타입의 클래스를 상속받아 PlatformLruCache 클래스를 정의하여 Picasso 객체가 사용되는 동안의 이미지들을 캐싱하고 있습니다. LruCache클래스는 [LastRecentlyUsed Cache](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU)) 알고리즘을 적용한 캐싱 클래스로 요소들을 새로 추가하거나 호출했을 경우 그 대상을 우선순위의 맨 앞으로 지정하고 만약 캐싱 크기가 꽉 찼을때 새로운 요소가 캐싱되어야 할 경우 우선순위가 제일 낮은 요소를 제거하는 로직을 사용하고 있습니다. 따라서 Picasso의 In-memory Cache는 자주 호출하는 이미지들이 더 빠른 속도로 호출되는 구조를 띄고 있습니다.
Picasso는 이 In-memory Cache의 용량을 내부의 Utils.calculateMemoryCacheSize 메서드를 호출하여 크기를 할당하는데 어플리케이션의 매니페스트 파일에 largeHeap 여부에 따라 다른 메서드를 호출하여 어플리케이션이 사용가능한 메모리 크기를 구한 후 연산을 하여 사용가능한 메모리의 15% 정도까지를 캐시 영역으로 할당해주고 있습니다. 캐시 생성 코드는 다음과 같습니다.
```java
public Picasso build() {
      //...other code
      
      if (cache == null) {
        cache = new PlatformLruCache(Utils.calculateMemoryCacheSize(context));
      }
      
      //...other code
```
```java
static int calculateMemoryCacheSize(Context context) {
    ActivityManager am = ContextCompat.getSystemService(context, ActivityManager.class);
    //largeHeap은 AndroidManifest의 <application> 내부의 largeHeap = true, largeHeap = false를 통해서 설정할 수 있습니다.
    boolean largeHeap = (context.getApplicationInfo().flags & FLAG_LARGE_HEAP) != 0;
    int memoryClass = largeHeap ? am.getLargeMemoryClass() : am.getMemoryClass();
    // Target ~15% of the available heap.
    return (int) (1024L * 1024L * memoryClass / 7);
  }
```
```java
PlatformLruCache(int maxByteCount) {
    cache = new LruCache<String, BitmapAndSize>(maxByteCount != 0 ? maxByteCount : 1) {
      @Override protected int sizeOf(String key, BitmapAndSize value) {
        return value.byteCount;
      }
    };
  }
```
핵심 코드는 가운데 블록의 calculateMemoryCacheSize() 메서드고 안드로이드 기기에서 실행되고 있는 프로세스들의 정보를 얻을 수 있는 ActivityManager 클래스를 통해서 현재 앱의 메모리를 계산 후 MB 단위로 변환하여 PlatformLruCache를 생성하게 됩니다.

Storage Cache는 Picasso를 활용하여 네트워크에서 이미지를 받아올 경우 이를 재사용할 때 네트워크 통신을 재호출하지 않고 로컬에 이미지를 저장하여 이미지 로딩 속도를 향상시키는데 사용합니다. Picasso는 네트워크에서 이미지를 불러올 때 [OkHttp3](https://square.github.io/okhttp/)  라이브러리를 사용하는데 OkHttp3 객체를 생성할 때 캐시 관련 옵션을 설정할 수 있습니다. okhttp3.Cache() 객체는 캐싱할 파일 저장 경로와 사이즈를 인자로 받아 생성하며 생성된 후 OkHttpClient.Builder().cache() 메서드에 인자로 넣어주면 OkHttp3를 사용하여 네트워크 요청을 할 시 해당 경로로 캐싱이 이루어집니다. okhttp3.Cache 객체는 PlatformLruCache 클래스와 마찬가지로 LRU Cache 알고리즘을 적용하여 캐싱이 이루어 지도록 되어있고 이를 구현하기 위해 JakeWarton이 개발한 [DiskLruCache](https://github.com/JakeWharton/DiskLruCache) 라이브러리를 사용하고 있습니다. 관련 코드는 다음과 같습니다.
```java
//in Picasso build()
//...other code

okhttp3.Cache unsharedCache = null;
      if (callFactory == null) {
        File cacheDir = createDefaultCacheDir(context);
        long maxSize = calculateDiskCacheSize(cacheDir);
        unsharedCache = new okhttp3.Cache(cacheDir, maxSize);
        callFactory = new OkHttpClient.Builder()
            .cache(unsharedCache)
            .build();
      }
      
//...other code
```
```java
//캐싱 폴더가 없을 경우 PICASSO_CACHE(폴더 이름 : "picasso-cache")를 이름으로 한 폴더 생성
static File createDefaultCacheDir(Context context) {
    File cache = new File(context.getApplicationContext().getCacheDir(), PICASSO_CACHE);
    if (!cache.exists()) {
      //noinspection ResultOfMethodCallIgnored
      cache.mkdirs();
    }
    return cache;
  }

//5~50MB 사이로 캐싱 폴더 사이즈를 조절
  static long calculateDiskCacheSize(File dir) {
    long size = MIN_DISK_CACHE_SIZE;

    try {
      StatFs statFs = new StatFs(dir.getAbsolutePath());
      //noinspection deprecation
      long blockCount =
          SDK_INT < JELLY_BEAN_MR2 ? (long) statFs.getBlockCount() : statFs.getBlockCountLong();
      //noinspection deprecation
      long blockSize =
          SDK_INT < JELLY_BEAN_MR2 ? (long) statFs.getBlockSize() : statFs.getBlockSizeLong();
      long available = blockCount * blockSize;
      // Target 2% of the total space.
      size = available / 50;
    } catch (IllegalArgumentException ignored) {
    }

    // Bound inside min/max size for disk cache.
    return Math.max(Math.min(size, MAX_DISK_CACHE_SIZE), MIN_DISK_CACHE_SIZE);
  }
```
Storage Cache에서도 눈여겨 볼 메서드는 calculateDiskCacheSize() 입니다. Android에서는 파일시스템 관련 정보를 얻기 위해 Statfs 클래스를 사용하는데 이를 통해 해당 캐싱 폴더 내의 블록 갯수와 블록 별 크기를 얻어 전체 크기를 구합니다. 그리고 이를 50으로 나눠 전체 캐싱 폴더에서 2% 정도를 사용할 수 있도록 하고 Math.max와 Math.min 연산을 통해 MIN_DISK_CACHE_SIZE와 MAX_DISK_CACHE_SIZE 를 넘지 않도록 하여 결과값을 반환해 줍니다. 나머지 두 메서드는 간단하여 소스와 참고 주석을 보시면 되겠습니다.
Picasso는 이와 같이 객체 생성 간 특별한 캐싱 옵션이 없어도 자동으로 In-memory Cache와 Storage Cache가 이루어져 이미지를 효율적으로 호출, 처리 할 수 있습니다. Android에서 이미지를 처리해야 될 경우 캐싱, 리사이징을 효과적으로 지원해주는 Picasso 라이브러리를 사용할 것을 적극 추천해드립니다.