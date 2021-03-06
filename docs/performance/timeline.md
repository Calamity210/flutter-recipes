---
title: Timeline
parent: Performance
nav_order: 2
wip: true
---

# Timeline

The most useful tool for analyzing performance is the Observatory timeline, you can use it to quickly locate and
diagnose problematic frames.

Unfortunately the observatory button was removed by the Flutter plugin, see
[IntelliJ Observatory](/docs/faq/intellij-observatory) for an easy way to open it again.

Keep in mind timelines can be skewed by external factors such as low memory, it's always good to double check by
enabling performance overlays in the developer options.

---

## Why not DevTools?

The DevTools timeline is also nice, but it has some limitations like not being able to get the class name of a sample.
It also lacks the ability to analyze live objects, which is extremely helpful for tracking down memory leaks.

Until DevTools has feature parity, I would recommend learning how to use the Observatory instead.

---

## Workflow

My general workflow goes as follows:

1.  Run the app in profile mode.
2.  Open the observatory.
3.  Click "view timeline" under VM information.
   
    ![](https://i.tst.sh/06Hvq.png)
4.  Click "Flutter Developer":

    ![](https://i.tst.sh/2rRlw.png)
5.  Prepare the device to produce problematic frames e.g. open the ListView.
6.  Click clear in the top right.

    ![](https://i.tst.sh/8K9Qj.png)
7.  Perform the action that produces problematic frames.
8.  Click refresh.
9.  Use the pan and zoom tools to locate a bad frame:
   
    ![](https://i.tst.sh/6irRQ.png)
   
    Make sure you scroll vertically to the ui and raster threads, bad frames will stick out like this:
   
    ![](https://i.tst.sh/m3oj7.png)
10. For ui thread problems like shown above, click the timeline event and then find the overlapping samples:
    
    ![](https://i.tst.sh/dBZ5S.png)
11. Locate problematic functions in the samples below:
    
    ![](https://i.tst.sh/WYvPe.png)
    
    Here we can see the root cause of the jank, I'm doing too much work in a closure in `_HomePageState.build`.

---

## Complex widgets

Unfortunately the culprit might not be a single function, but instead emerge from large complex widgets.

TODO