---
title: '解决Linux下Sublime text 3的中文输入问题 '
date: 2016-04-09 14:14:05
draft: true
---
http://c4fun.cn/blog/2013/11/30/linux-sublimetext-chinese/

搬个链接算了，博主已经写的很详细，就不重复编写了，只是为了留个记录。
基本方法就是这个，个别问题 可以google。

-------------------------------------------------
### 更新方法（编译部分与博主相同，懒得点的看下面）

新建文件sub-fcitx.c，建议放在Sublime Text的所在目录下，将下面的代码复制进去
```python

/*sublime-imfix.cUse LD_PRELOAD to interpose some function to fix sublime input method support for linux.By Cjacker Huang gcc -shared -o libsublime-imfix.so sublime-imfix.c `pkg-config --libs --cflags gtk+-2.0` -fPICLD_PRELOAD=./libsublime-imfix.so subl*/#include <gtk/gtk.h>#include <gdk/gdkx.h>typedef GdkSegment GdkRegionBox; struct _GdkRegion{ long size; long numRects; GdkRegionBox *rects; GdkRegionBox extents;}; GtkIMContext *local_context; voidgdk_region_get_clipbox (const GdkRegion *region, GdkRectangle *rectangle){ g_return_if_fail (region != NULL); g_return_if_fail (rectangle != NULL);  rectangle->x = region->extents.x1; rectangle->y = region->extents.y1; rectangle->width = region->extents.x2 - region->extents.x1; rectangle->height = region->extents.y2 - region->extents.y1; GdkRectangle rect; rect.x = rectangle->x; rect.y = rectangle->y; rect.width = 0; rect.height = rectangle->height; //The caret width is 2; //Maybe sometimes we will make a mistake, but for most of the time, it should be the caret. if(rectangle->width == 2 && GTK_IS_IM_CONTEXT(local_context)) { gtk_im_context_set_cursor_location(local_context, rectangle); }} //this is needed, for example, if you input something in file dialog and return back the edit area//context will lost, so here we set it again. static GdkFilterReturn event_filter (GdkXEvent *xevent, GdkEvent *event, gpointer im_context){ XEvent *xev = (XEvent *)xevent; if(xev->type == KeyRelease && GTK_IS_IM_CONTEXT(im_context)) { GdkWindow * win = g_object_get_data(G_OBJECT(im_context),"window"); if(GDK_IS_WINDOW(win)) gtk_im_context_set_client_window(im_context, win); } return GDK_FILTER_CONTINUE;} void gtk_im_context_set_client_window (GtkIMContext *context, GdkWindow *window){ GtkIMContextClass *klass; g_return_if_fail (GTK_IS_IM_CONTEXT (context)); klass = GTK_IM_CONTEXT_GET_CLASS (context); if (klass->set_client_window) klass->set_client_window (context, window);  if(!GDK_IS_WINDOW (window)) return; g_object_set_data(G_OBJECT(context),"window",window); int width = gdk_window_get_width(window); int height = gdk_window_get_height(window); if(width != 0 && height !=0) { gtk_im_context_focus_in(context); local_context = context; } gdk_window_add_filter (window, event_filter, context);}
```

#### 可能需要的模块

     build-essential  libgtk2.0-dev

切换到sub-fcitx.c，所在目录，编译生成so文件
```
gcc -shared -o libsublime-imfix.so sub-fcitx.c `pkg-config --libs --cflags gtk+-2.0` -fPIC
```
------------------------------
### 该方法在archlinux下 sublime_text 3 测试成功

编译好模块后，只要将原来的/usr/bin/subl3软链接删除或者改成一个脚本
```
#!/bin/sh
LD_PRELOAD=/opt/sublime_text_3/libsublime-imfix.so exec     /opt/sublime_text_3/sublime_text "$@"
```
模块以及软件目录根据自己的实际情况替换，即可实现从命令行以及桌面启动均能实现中文输入。