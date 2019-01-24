WHAT IT DOES?
=============
This is a  test APK for ContentProvider related testing.
The moudle tag is tests.

HOW IT WAS BUILT?
==================
[Build Method]

mmm vendor/mediatek/proprietary/frameworks/base/tests/ContentProvider

HOW TO USE IT?
==============
adb shell am instrument -e class com.mediatek.contentProvider.test.ContentProviderTest -w com.mediatek.contentProvider.test/android.test.InstrumentationTestRunner 
