## Testing Code Quality and Build Settings of Android Apps

### Verifying That the App is Properly Signed

#### Overview

Android requires that all APKs be digitally signed with a certificate before they can be installed. The digital signature is required by the Android system before installing/running an application, and it's also used to verify the identity of the owner for future updates of the application. This process can prevent an app from being tampered with, or modified to include malicious code.

When an APK is signed, a public-key certificate is attached to the APK. This certificate uniquely associates the APK to the developer and their corresponding private key. When building an app in debug mode, the Android SDK signs the app with a debug key specifically created for debugging purposes. An app signed with a debug key is not be meant for distribution and won't be accepted in most app stores, including the Google Play Store. To prepare the app for final release, the app must be signed with a release key belonging to the developer.

The final release build of an app must be signed with a valid release key. Note that Android expects any updates to the app to be signed with the same certificate, so a validity period of 25 years or more is recommended. Apps published on Google Play must be signed with a certificate that is valid at least until October 22th, 2033.

Two APK signing schemes are available: JAR signing (v1 scheme) APK Signature Scheme v2 (v2 scheme). The v2 signature, which is supported by Android 7.0 and higher, offers improved security and performance. Release builds should always be signed using *both* schemes.

#### Static Analysis

Verify that the release build is signed with both v1 and v2 scheme, and that the code signing certificate contained in the APK is belongs to the developer.

If you don't have the APK available locally, pull it from the device first:

```bash
$ adb shell pm list packages
(...)
package:com.awesomeproject
(...)
$ adb shell pm path com.awesomeproject
package:/data/app/com.awesomeproject-1/base.apk
$ adb pull /data/app/com.awesomeproject-1/base.apk
```

APK signatures can be verified using the <code>apksigner</code> tool.

```bash
$ apksigner verify --verbose Desktop/example.apk
Verifies
Verified using v1 scheme (JAR signing): true
Verified using v2 scheme (APK Signature Scheme v2): true
Number of signers: 1
```

The contents of the signing certificate can be examined using <code>jarsigner</code>. Note the in the debug certificate, the Common Name(CN) attribute is set to "Android Debug".

The output for an APK signed with a Debug certificate looks as follows:

```
$ jarsigner -verify -verbose -certs example.apk

sm     11116 Fri Nov 11 12:07:48 ICT 2016 AndroidManifest.xml

      X.509, CN=Android Debug, O=Android, C=US
      [certificate is valid from 3/24/16 9:18 AM to 8/10/43 9:18 AM]
      [CertPath not validated: Path does not chain with any of the trust anchors]
(...)
```

The output for an APK signed with a Release certificate looks as follows:

```
$ jarsigner -verify -verbose -certs example.apk

sm     11116 Fri Nov 11 12:07:48 ICT 2016 AndroidManifest.xml

      X.509, CN=Awesome Corporation, OU=Awesome, O=Awesome Mobile, L=Palo Alto, ST=CA, C=US
      [certificate is valid from 9/1/09 4:52 AM to 9/26/50 4:52 AM]
      [CertPath not validated: Path does not chain with any of the trust anchors]
(...)
```

Ignore the "CertPath not validated" error -  this error appears with Java SDK 7 and greater. Instead, you can rely on the <code>apksigner</code> to verify the certificate chain.

#### Dynamic Analysis

Static analysis should be used to verify the APK signature.

#### Remediation

Developers need to make sure that release builds are signed with the appropriate certificate from the release keystore. In Android Studio, this can be done manually or by configuring creating a signing configuration and assigning it to the release build type<sup>[2]</sup>.
The signing configuration can be managed through the Android Studio GUI or the <code>signingConfigs {}</code> block in <code>build.gradle</code>. The following values need to be set to activate both v1 and v2 scheme:

```
v1SigningEnabled true
v2SigningEnabled true
```

#### References

##### OWASP Mobile Top 10 2016
* M7 - Client Code Quality - https://www.owasp.org/index.php/Mobile_Top_10_2016-M7-Poor_Code_Quality

##### OWASP MASVS
- V7.1: "The app is signed and provisioned with valid certificate."

##### CWE
N/A

##### Info
- [1] Configuring your application for release - http://developer.android.com/tools/publishing/preparing.html#publishing-configure
- [2] Application Signing - https://developer.android.com/studio/publish/app-signing.html

##### Tools
- jarsigner - http://docs.oracle.com/javase/7/docs/technotes/tools/windows/jarsigner.html



### Testing If the App is Debuggable

#### Overview

The <code>android:debuggable</code> attribute in the <code>Application</code> tag in the Manifest determines whether or not the app can be debugged when running on a user mode build of Android. In a release build, this attribute should always be set to "false" (the default value).

#### Static Analysis

Check in <code>AndroidManifest.xml</code> whether the <code>android:debuggable</code> attribute is set:

```xml
<?xml version="1.0" encoding="utf-8" standalone="no"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.android.owasp">

    ...

    <application android:allowBackup="true" android:debuggable="true" android:icon="@drawable/ic_launcher" android:label="@string/app_name" android:theme="@style/AppTheme">
        <meta-data android:name="com.owasp.main" android:value=".Hook"/>
    </application>
</manifest>
```

#### Dynamic Analysis

Drozer can be used to identify if an application is debuggable. The module `app.package.attacksurface` displays information about IPC components exported by the application, in addition to whether the app is debuggable.

```
dz> run app.package.attacksurface com.mwr.dz
Attack Surface:
  1 activities exported
  1 broadcast receivers exported
  0 content providers exported
  0 services exported
    is debuggable
```

To scan for all debuggable applications on a device, the `app.package.debuggable` module should be used: 

```
dz> run app.package.debuggable 
Package: com.mwr.dz
  UID: 10083
  Permissions:
   - android.permission.INTERNET
Package: com.vulnerable.app
  UID: 10084
  Permissions:
   - android.permission.INTERNET
``` 

If an application is debuggable, it is trivial to get command execution in the context of the application. In `adb` shell, execute the `run-as` binary, followed by the package name and command:

```
$ run-as com.vulnerable.app id
uid=10084(u0_a84) gid=10084(u0_a84) groups=10083(u0_a83),1004(input),1007(log),1011(adb),1015(sdcard_rw),1028(sdcard_r),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats) context=u:r:untrusted_app:s0:c512,c768
```

An alternative method to determine if an application is debuggable, is to attach jdb to the running process. If debugging is disabled, this should fail with an error.

#### Remediation

In the `AndroidManifest.xml` file, set the `android:debuggable` flag to false, as shown below:

```xml
<application android:debuggable="false">
...
</application>
```

#### References

##### OWASP Mobile Top 10 2016
* M7 - Client Code Quality - https://www.owasp.org/index.php/Mobile_Top_10_2016-M7-Poor_Code_Quality

##### OWASP MASVS
* V7.2: "The app has been built in release mode, with settings appropriate for a release build (e.g. non-debuggable)."

##### CWE


-- TODO [Add relevant CWE for "Testing If the App is Debuggable"] --
* CWE-312 - Cleartext Storage of Sensitive Information

##### Info
* [1] Application element - https://developer.android.com/guide/topics/manifest/application-element.html

##### Tools

* Drozer - https://github.com/mwrlabs/drozer

### Testing for Debugging Symbols

#### Overview

-- TODO [Give an overview about the functionality and it's potential weaknesses] --

For native binaries, use a standard tool like nm or objdump to inspect the symbol table. A release build should generally not contain any debugging symbols. If the goal is to obfuscate the library, removing unneeded dynamic symbols is also recommended.

#### Static Analysis

Symbols  are usually stripped during the build process, so you need the compiled byte-code and libraries to verify whether the any unnecessary metadata has been discarded.

To display debug symbols:

```bash
export $NM = $ANDROID_NDK_DIR/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin/arm-linux-androideabi-nm
```

```bash
$ $NM -a libfoo.so
/tmp/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin/arm-linux-androideabi-nm: libfoo.so: no symbols
```
To display dynamic symbols:

```bash
$ $NM -D libfoo.so
```

Alternatively, open the file in your favorite disassembler and check the symbol tables manually.

#### Dynamic Analysis

Static analysis should be used to verify for debugging symbols.

#### Remediation

Dynamic symbols can be stripped using the <code>visibility</code> compiler flag. Adding this flag causes gcc to discard the function names while still preserving the names of functions declared as <code>JNIEXPORT</code>.

Add the following to build.gradle:

```
        externalNativeBuild {
            cmake {
                cppFlags "-fvisibility=hidden"
            }
        }
```

#### References

##### OWASP Mobile Top 10 2016
* M7 - Client Code Quality - https://www.owasp.org/index.php/Mobile_Top_10_2016-M7-Poor_Code_Quality

##### OWASP MASVS
* V7.3: "Debugging symbols have been removed from native binaries."

##### CWE

-- TODO [Add relevant CWE for "Testing for Debugging Symbols"] --
- CWE-312 - Cleartext Storage of Sensitive Information

##### Info

[1] Configuring your application for release - http://developer.android.com/tools/publishing/preparing.html#publishing-configure
[2] Debugging with Android Studio - http://developer.android.com/tools/debugging/debugging-studio.html

##### Tools

-- TODO [Add relevant tools for "Testing for Debugging Symbols"] --
* Enjarify - https://github.com/google/enjarify



### Testing for Debugging Code and Verbose Error Logging

#### Overview

`StrictMode` - https://developer.android.com/reference/android/os/StrictMode.html
-- TODO [Give an overview about the functionality and it's potential weaknesses] --

#### Static Analysis

-- TODO [Add content on white-box testing for "Testing for Debugging Code and Verbose Error Logging"] --

#### Dynamic Analysis

-- TODO [Add content on black-box testing for "Testing for Debugging Code and Verbose Error Logging"] --

#### Remediation

-- TODO [Describe the best practices that developers should follow to prevent this issue "Testing for Debugging Code and Verbose Error Logging"] --

#### References

##### OWASP Mobile Top 10 2016
* M7 - Client Code Quality - https://www.owasp.org/index.php/Mobile_Top_10_2016-M7-Poor_Code_Quality

##### OWASP MASVS
* V7.4: "Debugging code has been removed, and the app does not log verbose errors or debugging messages."

##### CWE
-- TODO [Add relevant CWE for "Testing for Debugging Code and Verbose Error Logging"] --
- CWE-312 - Cleartext Storage of Sensitive Information

##### Info
-- TODO

##### Tools
-- TODO [Add relevant tools for "Testing for Debugging Code and Verbose Error Logging"] --
* Enjarify - https://github.com/google/enjarify



### Testing Exception Handling

#### Overview

-- TODO [Give an overview about the functionality and it's potential weaknesses] --

#### Static Analysis

Review the source code to understand and identify how the application handles various types of errors (IPC communications, remote services invocation, etc). Here are some examples of the checks to be performed at this stage :

* Verify that the application use a well-designed and unified scheme to handle exceptions<sup>[1]</sup>.
* Verify that the application doesn't expose sensitive information while handling exceptions, but are still verbose enough to explain the issue to the user.

#### Dynamic Analysis

-- TODO [Describe how to test for this issue using static and dynamic analysis techniques. This can include everything from simply monitoring aspects of the app’s behavior to code injection, debugging, instrumentation, etc. ] --

#### Remediation

-- TODO [Describe the best practices that developers should follow to prevent this issue "Testing Exception Handling"] --

#### References

##### OWASP Mobile Top 10 2016
* M7 - Client Code Quality - https://www.owasp.org/index.php/Mobile_Top_10_2016-M7-Poor_Code_Quality

##### OWASP MASVS
* V7.5: "The app catches and handles possible exceptions."
* V7.6: "Error handling logic in security controls denies access by default."

##### CWE
-- TODO [Add relevant CWE for "Testing Exception Handling"] --
- CWE-312 - Cleartext Storage of Sensitive Information

##### Info

[1] Exceptional Behavior (ERR) - https://www.securecoding.cert.org/confluence/pages/viewpage.action?pageId=18581047

##### Tools

-- TODO [Add relevant tools for "Testing Exception Handling"] --
* Enjarify - https://github.com/google/enjarify




### Testing for Memory Bugs in Unmanaged Code

#### Overview

-- TODO [Give an overview about the functionality and it's potential weaknesses] --

#### Static Analysis

-- TODO [Add content for white-box testing "Testing for Memory Management Bugs"] --

#### Dynamic Analysis

-- TODO [Add content for black-box testing "Testing for Memory Management Bugs"] --

#### Remediation

-- TODO [Add remediations for "Testing for Memory Management Bugs"] --

#### References

##### OWASP Mobile Top 10 2016
* M7 - Client Code Quality - https://www.owasp.org/index.php/Mobile_Top_10_2016-M7-Poor_Code_Quality

##### OWASP MASVS
* V7.7: "In unmanaged code, memory is allocated, freed and used securely."

##### CWE
-- TODO [Add relevant CWE for "Testing for Memory Management Bugs"] --
- CWE-312 - Cleartext Storage of Sensitive Information

##### Info
* Configuring your application for release - http://developer.android.com/tools/publishing/preparing.html#publishing-configure
* Debugging with Android Studio - http://developer.android.com/tools/debugging/debugging-studio.html

##### Tools
-- TODO [Add relevant tools for "Testing for Memory Management Bugs"] --
* Enjarify - https://github.com/google/enjarify



### Verify That Free Security Features Are Activated

#### Overview

As Java classes are trivial to decompile, applying some basic obfuscation to the release bytecode is recommended. For Java apps on Android, ProGuard offers an easy way to shrink and obfuscate code. It replaces identifiers such as class names, method names and variable names with meaningless character combinations. This is a form of layout obfuscation, which is “free” in that it doesn't impact the performance of the program.

Since most Android applications are Java based, they are immune<sup>[1]</sup> to buffer overflow vulnerabilities.


#### Static Analysis

If source code is provided, the build.gradle file can be checked to see if obfuscation settings are applied. From the example below, you can see that `minifyEnabled` and `proguardFiles` are set. It is common to create exceptions for some classes from obfuscation with "-keepclassmembers" and "-keep class". Therefore it is important to audit the ProGuard configuration file to see what classes are exempted. The `getDefaultProguardFile('proguard-android.txt')` method gets the default ProGuard settings from the `<Android SDK>/tools/proguard/` folder. The file `proguard-rules.pro` is where you define custom ProGuard rules. From our sample `proguard-rules.pro` file, you can see that many classes that extend common android classes are exempted, which should be done more granular on specific classes or libraries.

build.gradle
```
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                    'proguard-rules.pro'
        }
    }
    ...
}
```

proguard-rules.pro
```
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Application
-keep public class * extends android.app.Service
```

#### Dynamic Analysis

If source code is not provided, an APK can be decompiled to verify if the codebase has been obfuscated. dex2jar can be used to convert dex code to jar file. Tools like JD-GUI can be used to check if class, method and variable name is human readable.

Sample obfuscated code block
```
package com.a.a.a;

import com.a.a.b.a;
import java.util.List;

class a$b
  extends a
{
  public a$b(List paramList)
  {
    super(paramList);
  }

  public boolean areAllItemsEnabled()
  {
    return true;
  }

  public boolean isEnabled(int paramInt)
  {
    return true;
  }
}
```

#### Remediation

ProGuard should be used to strip unneeded debugging information from the Java bytecode. By default, ProGuard removes attributes that are useful for debugging, including line numbers, source file names and variable names. ProGuard is a free Java class file shrinker, optimizer, obfuscator and pre-verifier. It is shipped with Android’s SDK tools. To activate shrinking for the release build, add the following to build.gradle:

```
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile(‘proguard-android.txt'),
                    'proguard-rules.pro'
        }
    }
    ...
}
```

#### References

##### OWASP Mobile Top 10 2016
* M7 - Client Code Quality - https://www.owasp.org/index.php/Mobile_Top_10_2016-M7-Poor_Code_Quality

##### OWASP MASVS
* V7.8: "Free security features offered by the toolchain, such as byte-code minification, stack protection, PIE support and automatic reference counting, are activated."

##### CWE
-- TODO [Add relevant CWE for Verifying that Java Bytecode Has Been Minified] --
- CWE-312 - Cleartext Storage of Sensitive Information

##### Info
[1] Java Buffer Overflows - https://www.owasp.org/index.php/Reviewing_Code_for_Buffer_Overruns_and_Overflows#.NET_.26_Java
[2] Configuring your application for release - http://developer.android.com/tools/publishing/preparing.html#publishing-configure
[3] Debugging with Android Studio - http://developer.android.com/tools/debugging/debugging-studio.html

##### Tools
-- TODO [Add relevant tools for Verifying that Java Bytecode Has Been Minified] --
* Enjarify - https://github.com/google/enjarify
