# Android图形系统

## requestLayout和invalidate

### invalidate

```java
void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
        boolean fullInvalidate) {
    if (skipInvalidate()) {
        return;
    }

    mCachedContentCaptureSession = null;

    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
            || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
            || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
            || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
        if (fullInvalidate) {//fullInvalidate=true
            mLastIsOpaque = isOpaque();
            mPrivateFlags &= ~PFLAG_DRAWN;//1.移除PFLAG_DRAWN
        }

        mPrivateFlags |= PFLAG_DIRTY;//2.增加PFLAG_DIRTY

        if (invalidateCache) {//invalidateCache=true
            mPrivateFlags |= PFLAG_INVALIDATED;//3.增加PFLAG_INVALIDATED
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;//4.清除PFLAG_DRAWING_CACHE_VALID
        }

        final AttachInfo ai = mAttachInfo;
        final ViewParent p = mParent;
        if (p != null && ai != null && l < r && t < b) {
            final Rect damage = ai.mTmpInvalRect;
            damage.set(l, t, r, b);
            p.invalidateChild(this, damage);//5.向parent传播脏区damage
        }
    }
}
```

