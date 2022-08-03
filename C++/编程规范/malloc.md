# malloc

```cpp
#ifndef SECUREC_MALLOC
#define SECUREC_MALLOC(x) malloc((size_t)(x))
#endif#ifndef SECUREC_FREE
#define SECUREC_FREE(x) free((void *)(x))
#endif
```

```C++
void *DmsdpMalloc(int32_t moduleId, uint32_t size)
{
    void *memBuf = (void *)SECUREC_MALLOC(size);
    if (memBuf == NULL) {
        LOGE("moduleId %d malloc failed", moduleId);
        return NULL;
    }
#if DMSDP_MEMORY_DEBUG
    int32_t lockRet = 0;
    if (!g_isInit) {
        lockRet = DMSDPMutexInit(&g_memLock);
        if (lockRet != 0) {
            LOGE("memory mutex init failed , error %d", lockRet);
            return memBuf;
        }
        g_isInit = true;
    }    DMSDPMutexLock(&g_memLock);
    int32_t i;
    for (i = 0; i < MAX_MEMORY_NODE_SIZE; i++) {
        if (!g_memoryInfoList[i].used) {
            g_memoryInfoList[i].used = true;
            g_memoryInfoList[i].size = size;
            g_memoryInfoList[i].buffer = memBuf;
            g_memoryInfoList[i].moduleId = moduleId;
            break;
        }
    }
    DMSDPMutexUnlock(&g_memLock);
    LOGD("moudleId %d malloc sucess, memBuf %p size %d", moduleId, memBuf, size);
#endif
    return memBuf;
}
```

---
