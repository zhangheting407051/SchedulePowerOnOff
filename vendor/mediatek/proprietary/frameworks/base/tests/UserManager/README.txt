Test cases stubs for android frameworks UserManager module

WHAT IT DOES?
=============
MediaTekUserManagerTest.apk is a test case stubs for usermanager
test cases.

It provide test stubs for MediaTekUserManagerTest.apk

The test cases are most come from Android CTS test cases

HOW IT WAS BUILT?
==================
make 'vendor/mediatek/proprietary/frameworks/base/tests/UserManager/' folder to get
test apk.

HOW TO USE IT?
==============
After build MediaTekUserManagerTest.apk, install to phone with
MediaTekUserManagerTest.apk, and run command:
'adb shell am instrument -w com.mediatek.cts.usermanager/android.test.InstrumentationTestRunner'
to get result.
