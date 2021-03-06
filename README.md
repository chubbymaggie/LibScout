# LibScout

LibScout is a light-weight and effective static analysis tool to detect third-party libraries in Android/Java apps. The detection is resilient against common bytecode obfuscation techniques such as identifier renaming or code-based obfuscations such as reflection-based API hiding or control-flow randomization. Further LibScout is capable of pinpointing exact library versions including versions that contain severe bugs or security issues.<br>

LibScout requires the original library SDKs (compiled .jar/.aar files) to extract library profiles that can be used for detection on Android apps. Pre-generated library profiles are hosted at the repository [LibScout-Profiles](https://github.com/reddr/LibScout-Profiles).

Unique detection features:
 * Library detection resilient against many kinds of bytecode obfuscation (e.g. obfuscations by ProGuard)
 * Capability of pinpointing the exact library version (in some cases to a set of 2-3 candidate versions)
 * Capability of handling dead-code elimination, by computing a similarity score against baseline SDKs
 * Library API usage analysis upon detection

Besides detecting libraries in apps, LibScout conducts API analyses on the library SDKs to determine API compatibility across versions and adherence to semantic versioning.
The resulting data has recently been used to build the Android Studio extension [up2dep](https://github.com/ngcuongst/up2dep) to help developers keeping their dependencies up-to-date.


## Library History Scraper (./scripts)

The scripts directory contains python scripts to automatically download original library SDKs including complete version histories from *Maven Central*, *JCenter* and *custom mvn repositories*. The original library SDKs can be used to generate profiles and to conduct library API analyses (see modules below). Use the [library-profile-generator](scripts/library-profile-generator.sh) script to conveniently generate profiles at scale.

The scrapers need to be configured with a json config that includes metadata of the libraries to be fetched (name, groupid, artefactid). There is currently a config file for mvn central with over 100 libraries and a config to download Android libraries from Google's maven repository (350 libraries, including support, gms, ktx, jetpack, ..).


## Detecting (vulnerable) library versions
Ready-to-use library profiles and library meta-data can be found in the repository [LibScout-Profiles](https://github.com/reddr/LibScout-Profiles).

LibScout has builtin functionality to report library versions with the following security vulnerabilities.<br>
The pre-generated profiles for vulnerable versions are tagged with <b>[SECURITY]</b>, patches with <b>[SECURITY-FIX]</b>. <br>
This information is encoded in the library.xml files that have been used to generate the profiles.
We try to update the list/profiles whenever we encounter new security issues. If you can share information, please let us know.


| Library    |   Version(s)    | Fix Version   |  Vulnerability                         |     Link  |
| ---------- | ---------------:|--------------:|--------------------------------------- | ---------------------------------------------------------------------------------------------------------------   |
| Airpush    |      < 8.1      |  > 8.1        |  Unsanitized default WebView settings  |  [Link](https://support.google.com/faqs/answer/6376737)  |
| Apache CC  | 3.2.1 / 4.0     |  3.2.2 / 4.1  |  Deserialization vulnerability         |  [Link](http://www.kb.cert.org/vuls/id/576313)  |
| Dropbox    | 1.5.4 - 1.6.1   |   1.6.2       |  DroppedIn vulnerability               |  [Link](https://blogs.dropbox.com/developers/2015/03/security-bug-resolved-in-the-dropbox-sdks-for-android)  |
| Facebook   |       3.15      |    3.16       |  Account hijacking vulnerability       |  [Link](http://thehackernews.com/2014/07/facebook-sdk-vulnerability-puts.html)  |
| MoPub      |    < 4.4.0      |  4.4.0        |  Unsanitized default WebView settings  |  [Link](https://support.google.com/faqs/answer/6345928)  |
| OkHttp     | 2.1 - 2.7.4 <br>3.0.0- 3.1.2  |  2.7.5<br>3.2.0  |  Certificate pinning bypass  |  [Link](https://medium.com/square-corner-blog/vulnerability-in-okhttps-certificate-pinner-2a7326ad073b)  |
| Plexus Archiver    |  < 3.6.0        |  3.6.0        | Zip Slip vulnerability                | [Link](https://github.com/snyk/zip-slip-vulnerability)
| SuperSonic |    < 6.3.5      |   6.3.5       |  Unsafe functionality exposure via JS  |  [Link](https://support.google.com/faqs/answer/7126517)  |
| Vungle     |    < 3.3.0      |  3.3.0        |  MitM attack vulnerability             |  [Link](https://support.google.com/faqs/answer/6313713)  |
| ZeroTurnaround | < 1.13      | 1.13          |  Zip Slip vulnerability                | [Link](https://github.com/snyk/zip-slip-vulnerability)

On our last scan of free apps on Google Play (05/25/2017), LibScout detected >20k apps containing one of these vulnerable lib versions.
These results have been reported to Google's [ASI program](https://developer.android.com/google/play/asi.html) (still under investigation).


##  LibScout 101

 * LibScout requires Java 1.8 or higher and can be build with Gradle.
 * Generate a runnable jar with the gradle wrapper <i>gradlew</i> (Linux/MacOS) or <i>gradlew.bat</i> (Windows), by invoking it with the <i>build</i> task, e.g. <i>./gradlew build</i><br>
   The <i>LibScout.jar</i> is output to the <i>build/libs</i> directory.
 * Most LibScout modules require an Android SDK (jar) to distinguish app code from framework code (via the -a switch).<br>
Refer to <a href="https://developer.android.com/studio/">https://developer.android.com/studio/</a> for download instructions.
 * By default, LibScout logs to stdout. Use the -d switch to redirect output to files. The -m switch disables any text output. Depending on the operation mode (see below), LibScout's results can be written to disk in JSON format or JAVA serialization.
 * LibScout repo structure in a nutshell:<br>
<pre><code>
|_ gradlew / gradlew.bat (gradle wrappers to generate runnable LibScout.jar)
|_ assets
|    |_ library.xml (Library meta-data template)
|_ data
|    |_ app-version-codes.csv (Google Play app packages with valid version codes)
|_ lib
|    pre-compiled WALA libs, Android axml
|_ logging
|    |_ logback.xml (log4j configuration file)
|_ scripts
|    |_ mvn-central (scraper for mvn-central)
|    |_ jcenter+mvn (scraper for jcenter/custom mvn repos)
|_ src
    source directory of LibScout (de/infsec/tpl). Includes some open-source,
    third-party code to parse AXML resources / app manifests etc.
</code></pre>
  * LibScout supports different use cases implemented as modules (modes of operation). Below a detailed description for each module.

### Library Profiling (-o profile)

This module generates unique library fingerprints from original lib SDKs (.jar and .aar files supported). These profiles can subsequently be used for testing whether the respective library
versions are included in apps. Each library file additionally requires a <i>library.xml</i> that contains meta data (e.g. name, version,..). A template can be found in the assets directory.
For your convenience, you can use our library scrapers (./scripts) that download full library histories from Maven repositories.
By default, LibScout generates hashtree-based profiles with Package and Class information (omitting methods).<br>
<pre>java -jar LibScout.jar -o profile -a <i>android_sdk</i> -x <i>path_to_library_xml</i> <i>path_to_library_file</i> </pre>

### Library Detection (-o match)

Detects libraries in apps using pre-generated profiles. Optionally, LibScout also conducts an API usage analysis for  detected libraries, i.e. which library APIs are used by the app or by other libraries (-u switch).<br>
Analysis results can be written in different formats.
<ol>
    <li> the JSON format (-j switch), creates subfolders in the specified directory following the app package, i.e. *com.foo* will create *com/foo* subfolders.
        This is useful when coping with a large number of apps. For detailed information about the information stored, please refer to the <a href="https://github.com/reddr/LibScout/wiki#json-output-format-specification">JSON output specification</a>.</li>
    <li> the <b>serialization</b> option (-s switch) writes stat files per app to disk that can be used with the database module to create a SQLite database (the DB structure can be found in class
    <b>de.infsec.tpl.stats.SQLStats</b></li>
</ol>
<pre>java -jar LibScout.jar -o match -a <i>android_sdk</i> -p <i>path_to_profiles</i> [-u] [-j <i>json_dir</i>] [-s <i>stats_dir</i>] [-m] [-d <i>log_dir</i>] <i>path_to_app(s)</i>  </pre>

### Database Generator (-o db)

Generates a SQLite database from library profiles and serialized app stats:<br>
<pre>java -jar LibScout.jar -o db -p <i>path_to_profiles</i> -s <i>path_to_app_stats</i> </pre>

### Library API analysis (-o lib_api_analysis)

Analyzes changes in the documented (public) API sets of library versions.<br>
The analysis results currently include the following information:

Compliance to <a href="http://semver.org">Semantic Versioning (SemVer)</a>, i.e. whether the change in the version string between consecutive versions (expected SemVer) matches
the changes in the respective public API sets (actual SemVer). Results further include statistics about changes in API sets (additions/removals/modifcations). For removed APIs,
LibScout additionally tries to infer alternative APIs (based on different features).<br>

For the analysis, you have to provide a path to library SDKs. LibScout recursively searches for library jars|aars (leaf directories are expected to have at most one jar|aar file and one library.xml file).
For your convenience use the Maven Central Scraper. Analysis results are written to disk in JSON format (-j switch).<br>
<pre>java -jar LibScout.jar -o lib_api_analysis -a <i>android_sdk</i> [-j <i>json_dir</i>] <i>path_to_lib_sdks</i></pre>

## Scientific Publications

For technical details and large-scale evaluation results, please refer to our publications:<br>
> Reliable Third-Party Library Detection in Android and its Security Applications<br>
> https://www.infsec.cs.uni-saarland.de/~derr/publications/pdfs/derr_ccs16.pdf
>
> Keep me Updated: An Empirical Study of Third-Party Library Updatability on Android<br>
> https://www.infsec.cs.uni-saarland.de/~derr/publications/pdfs/derr_ccs17.pdf

If you use LibScout in a scientific publication, we would appreciate citations using these Bibtex entries: [[bib-ccs16]](https://www.infsec.cs.uni-saarland.de/~derr/publications/bib/derr_ccs16.bib)
[[bib-ccs17]](https://www.infsec.cs.uni-saarland.de/~derr/publications/bib/derr_ccs17.bib)<br>


