Test cases stubs for android frameworks LoaderManager module

WHAT IT DOES?
=============
MediaTekLoaderManagerTest.apk is a test case stubs for loadermanager
test cases.

It provide test stubs for MediaTekLoaderManagerTest.apk

The test cases are most come from Android CTS test cases

HOW IT WAS BUILT?
==================
make 'vendor/mediatek/proprietary/frameworks/base/tests/LoaderManager/' folder to get
test apk.

HOW TO USE IT?
==============
After build MediaTekLoaderManagerTest.apk, install to phone with
MediaTekLoaderManagerTest.apk, and run command:
'adb shell am instrument -w com.mediatek.cts.loadermanager/android.test.InstrumentationTestRunner'
to get result.
