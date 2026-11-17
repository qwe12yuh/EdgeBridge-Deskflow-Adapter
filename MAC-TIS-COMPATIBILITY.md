# macOS keyboard input-source compatibility

A Deskflow 1.27.0 development build using Qt 6.12 aborted during startup on macOS. The crash stacks showed AppKit's main thread reading the keyboard layout for menu shortcuts while Deskflow's worker thread entered `TISCreateInputSourceList` from `OSXComputer` through `AppUtilUnix::getKeyboardLayoutList`. A lock inside Deskflow cannot serialize calls made independently by AppKit.

The patch dispatches Deskflow's Text Input Source access to the Qt main thread. Callers already on that thread execute directly. Worker-thread calls wait for the main thread's result. While waiting for the core worker to finish at shutdown, the main thread continues servicing queued method calls so that a final input-source lookup can complete.

`OSXTisThreadTests` reads the local keyboard layout without opening a window. It exercises a main-thread call, a worker-thread call and a call made during shutdown. These checks passed on the tested macOS and Qt combination. They cover startup and shutdown of input-source access. Native file-drop behavior is tested separately. [Build and test instructions](README.md)
