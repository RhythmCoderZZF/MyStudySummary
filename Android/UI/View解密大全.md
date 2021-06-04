# View解密大全

## requestLayout

Call this when something has changed which has invalidated the layout of this view. 

This will schedule a layout pass of the view  tree. 

注意：This should not be called while the view hierarchy is currently in a layout pass ({@link #isInLayout()}. If layout is happening, the request may be honored at the end of the current layout pass (and then layout will run again) or after the current frame is drawn and the next layout occurs.

