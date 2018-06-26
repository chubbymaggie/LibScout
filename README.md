# LibScout

LibScout is a light-weight and effective static analysis tool to detect third-party libraries in Android/Java apps. The detection is resilient against common bytecode obfuscation techniques such as identifier renaming or code-based obfuscations such as reflection-based API hiding or control-flow randomization. Further LibScout is capable of pinpointing exact library versions.<br>
LibScout requires the original library SDKs (compiled .jar/.aar files) to extract library profiles that can be used for detection on Android apps. Pre-generated library profiles are hosted at the repository [LibScout-Profiles](https://github.com/reddr/LibScout-Profiles).

Unique features:
 * Library detection resilient against many kinds of bytecode obfuscation
 * Capability of pinpointing the exact library version (in some cases to a set of 2-3 candidate versions)
 * Capability of handling dead-code elimination, by computing a similarity score against baseline SDKs

For technical details and large-scale evaluation results, please refer to our publications:<br>
> Reliable Third-Party Library Detection in Android and its Security Applications<br>
> https://www.infsec.cs.uni-saarland.de/~derr/publications/pdfs/derr_ccs16.pdf
>
> Keep me Updated: An Empirical Study of Third-Party Library Updatability on Android<br>
> https://www.infsec.cs.uni-saarland.de/~derr/publications/pdfs/derr_ccs17.pdf

If you use LibScout in a scientific publication, we would appreciate citations using these Bibtex entries: [[bib-ccs16]](https://www.infsec.cs.uni-saarland.de/~derr/publications/bib/derr_ccs16.bib)
[[bib-ccs17]](https://www.infsec.cs.uni-saarland.de/~derr/publications/bib/derr_ccs17.bib)<br>


##   Library Profiles and Scripts

Ready-to-use library profiles and library meta-data can be found in the repository [LibScout-Profiles](https://github.com/reddr/LibScout-Profiles).
It further includes scripts to automatically retrieve complete library version histories.

### Detecting vulnerable library versions

LibScout has builtin functionality to report library versions with the following security vulnerabilities.<br>
Detected vulnerable versions are tagged with <b>[SECURITY]</b>, patches with <b>[SECURITY-FIX]</b>. <br>
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

 * LibScout requires Java 1.8 or higher. A runnable jar can be generated with the ant script <i>build.xml</i>
 * Most LibScout modules require an Android SDK (jar) to distinguish app code from framework code (via the -a switch).<br>
Refer to <a href="https://developer.android.com/studio/">https://developer.android.com/studio/</a> for download instructions.
 * By default, LibScout logs to stdout. Use the -d switch to redirect output to files. The -m switch disables any text output.
 * A short view on the repo structure:<br>
<pre><code>
|_ build.xml (ant build file to generate runnable .jar)
|_ assets
|    |_ library.xml (Library meta-data template)
|_ data
|    |_ app-version-codes.csv (Google Play app packages with valid version codes)
|_ lib
|    pre-compiled WALA libs, Apache commons*, log4j
|_ logging
|    |_ logback.xml (log4j configuration file)
|_ src
    source directory of LibScout (de/infsec/tpl). Includes some open-source,
    third-party code to parse AXML resources / app manifests etc.
</code></pre>
  * LibScout supports different use cases implemented as modules (modes of operation). Below a detailed description for each module.

### Library Profiling (-o profile)

This module generates unique library fingerprints from original lib SDKs (.jar and .aar files supported). These profiles can subsequently be used for testing whether the respective library
versions are included in apps. Each library file additionally requires a <i>library.xml</i> that contains meta data (e.g. name, version,..). A template can be found in the assets directory.
For your convenience, you can use our ([Library Scraper](https://github.com/reddr/LibScout-Profiles/tree/master/scripts/mvn-central)) that downloads full library histories from Maven Central.
By default, LibScout generates hashtree-based profiles with Package and Class information (omitting methods).<br>
<pre>java -jar LibScout.jar -o profile -a lib/android-X.jar -x ${lib-dir/library.xml} ${lib-dir/lib.[jar|aar]} </pre>

### Library Detection (-o match)

Detects libraries in apps using pre-generated profiles. Analysis results can be written in to formats.
<ol>
    <li> the JSON format (-j switch), creates subfolders in the specified directory following the app package, i.e. com.foo will create com/foo subfolders.
        This is useful when coping with a large number of apps.</li>
    <li> the <b>serialization</b> option (-s switch) writes stat files per app to disk that can be used with the database module to create a SQLite database (the DB structure can be found in class
    <b>de.infsec.tpl.stats.SQLStats</b></li>
</ol>
The following example both logs to directory and serializes results to disk:<br>
<pre>java -jar LibScout.jar -o match -a lib/android-X.jar -p &lt;path-to-lib-profiles&gt; -s -d &lt;log-dir&gt; someapp.apk  </pre>

### Database Generator (-o db)

Generates a SQLite database from library profiles and serialized app stats:<br>
<pre>java -jar LibScout.jar -o db -p &lt;path-to-lib-profiles&gt; -s &lt;path-to-app-stats&gt; </pre>

### Library API analysis (-o lib_api_analysis)

Analyzes changes in the documented (public) API sets of library versions.<br>
The analysis results currently include the following information:

Compliance to <a href="http://semver.org">Semantic Versioning (SemVer)</a>, i.e. whether the change in the version string between consecutive versions (expected SemVer) matches
the changes in the respective public API sets (actual SemVer). Results further include statistics about changes in API sets (additions/removals/modifcations). For removed APIs,
LibScout additionally tries to infer alternative APIs (based on different features).<br>

For the analysis, you have to provide a path to library SDKs. LibScout recursively searches for library jars|aars (leaf directories are expected to have at most one jar|aar file and one library.xml file).
For your convenience use the Maven Central Scraper. Analysis results are written to disk in JSON format (-j switch).<br>
<pre>java -jar LibScout.jar -o lib_api_analysis -a lib/android-X.jar -j &lt;json-dir&gt; path-to-lib-sdks</pre>

