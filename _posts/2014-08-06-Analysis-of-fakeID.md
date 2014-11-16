---
layout: post
title: "Analysis of FakeID"
description: "An analysis of the Android FakeID bug by viewing the patch released by Google"
category: articles
tags: [Source review, Android, FakeID, hacking, Google bug 13678484]
modified: 2014-07-06
comments: false
share: false
---

When I heard about Google bug 13678484 (AKA Fake ID) I wanted to understand the issue and its implications, and as I am too impatient to wait until the Blackhat 2014 talks are made available I decided to do some research of my own. As a security consultant and self proclaimed hacker, i'm always interested in finding out details regarding new Android bugs with potential widespread impact. As I tend to spend a lot of time performing mobile security assessments, I also know how important it is to understand the new issues and threats in order to advise on how to protect mobile applications. At the time of writing, technical information on this bug has been sparse and hopefully this blog post will help in sharing more information about this attack and to provide some input on whether this bug is really as critical as current publications have lead us to believe.

Fake ID was discovered by Bluebox Labs and dates back to 2010 with Android 2.1, details of which will be presented at Blackhat USA 2014. Amongst the tweets advocating underage drinking, searching for #fakeid on twitter shows how the community is buzzing with talk regarding the latest critical vulnerability in Android. According to Bluebox Security, Fake ID allows malicious apps the ability to impersonate trusted applications on a device. This is due to an issue with Android's application signature checking in Android < 4.1. The Bluebox blog is a little hazy about which versions this bug has been patched, but mention that there is a possibility of a patch existing in 4.1, 4.2, 4.3 and 4.4. In the initial reports, Bluebox discussed using this vulnerability to exploit a webview in an application by creating a malicious application which impersonates Adobe. It should be noted, however that the Webview issue is separate to Fake ID and has been fixed in Android 4.4.

By searching for the bug number, we can find a link to the patch issued by [Google in April 2014](https://android.googlesource.com/platform/libcore/+/android-cts-4.1_r4) and can see that [JarFile.java](https://android.googlesource.com/platform/libcore/+/2bc5e811a817a8c667bca4318ae98582b0ee6dc6%5E!/#F0), [JarVerifier.java](https://android.googlesource.com/platform/libcore/+/2bc5e811a817a8c667bca4318ae98582b0ee6dc6%5E!/#F1) and [JarUtils.java](https://android.googlesource.com/platform/libcore/+/2bc5e811a817a8c667bca4318ae98582b0ee6dc6%5E!/#F2) have been modified. To understand the bug, well start by looking at the source code for JarFile and we find:

{% highlight java %}
1     public JarFile(File file, boolean verify, int mode) throws IOException {
2 +        this(file, verify, mode, false);
3 +    }
...
4     public JarFile(String filename, boolean verify) throws IOException {
5 +        this(filename, verify, false);
6 +    }
{% endhighlight %}

Above, at line 2 and line 5, we can see that the constructors have been modified to call another constructor, passing in an extra parameter; a boolean false. 

{% highlight java %}
1 +    public JarFile(File file, boolean verify, int mode, boolean chainCheck) throws IOException {
2         super(file, mode);
3         if (verify) {
4 -            verifier = new JarVerifier(file.getPath());
5 +            verifier = new JarVerifier(file.getPath(), chainCheck);
...
6  +    public JarFile(String filename, boolean verify, boolean chainCheck) throws IOException {
7         super(filename);
8           if (verify) {
9  -            verifier = new JarVerifier(filename);
10  +            verifier = new JarVerifier(filename, chainCheck);
{% endhighlight %}

Looking at the patch, we also see that two more constructor were added (called by the modified constructors), taking as arguments the arguments passed to the original constructors plus a chainCheck flag - shown at line 1 and line 7. Line 4,5 and 9,10 also show that this flag is now used when calling JarVerifier. For the curious, Java does not support default parameters in functions, so by making the changes as shown above, Google can fix the issue without having to modify the interface to JarFile.

In JarVerifier, we have the following changes:

{% highlight java %}
1 +    private final boolean chainCheck;
...
2  +    JarVerifier(String name) {
3  +        this(name, false);
4  +    }
...
5	-    JarVerifier(String name) {
6	+    JarVerifier(String name, boolean chainCheck) {
7	         jarName = name;
8	+        this.chainCheck = chainCheck;
9	     }
{% endhighlight %}

In the above snippet at line 6, we can see JarVerifier now takes an extra parameter, although a convenience constructor still exists should the function without the chainCheck parameter be used.

{% highlight java %}
1         try {
2             Certificate[] signerCertChain = JarUtils.verifySignature(
3                     new ByteArrayInputStream(sfBytes),
4 -                    new ByteArrayInputStream(sBlockBytes));
5 +                    new ByteArrayInputStream(sBlockBytes),
6 +                    chainCheck);
{% endhighlight %}

Again, we can see that chainCheck is used as an extra argument to JarUtils.verifySignature(). The first and second arguments are unchanged and are ByteArrayInputStreams created from byte arrays containing the data from the signature file and certificate file respectivly.

The final file to be updated to patch Fake ID is JarUtils.java

{% highlight java %}
1 +     * @see #verifySignature(InputStream, InputStream, boolean)
2 +     */
3 +    public static Certificate[] verifySignature(InputStream signature, InputStream signatureBlock)
4 +            throws IOException, GeneralSecurityException {
5 +        return verifySignature(signature, signatureBlock, false);
6 +    }
7 +
8 +    /**
...
9     public static Certificate[] verifySignature(InputStream signature, InputStream
10 -            signatureBlock) throws IOException, GeneralSecurityException {
11 +            signatureBlock, boolean chainCheck) throws IOException, GeneralSecurityException {
{% endhighlight %}

Similar to the other files, we see a modification to the verifySignature function on line 3 to provide a default parameter to an overloaded function which expects a boolean chainCheck argument seen at line 11.

{% highlight java %}
1     public static Certificate[] verifySignature(InputStream signature, InputStream
...
2  -        return createChain(certs[issuerSertIndex], certs);
3  +        return createChain(certs[issuerSertIndex], certs, chainCheck);
4       }
5  -    private static X509Certificate[] createChain(X509Certificate  signer, X509Certificate[] candidates) {
6  +    private static X509Certificate[] createChain(X509Certificate  signer,
7  +            X509Certificate[] candidates, boolean chainCheck) {
...
8           X509Certificate issuerCert;
9  +        X509Certificate subjectCert = signer;
...
10 +        issuerCert = findCert(issuer, candidates, subjectCert, chainCheck);
{% endhighlight %}

On line 3, line 7 and line 10 we can see how the chainCheck parameter is now passed to the createChain method, then onto the findCert method. On line 9 we can also see a new X509Certificate object is being created from the argument, signer (i.e. the certificate we're checking), to the createChain method. As the previous line contains an X509Certificate object called issuerCert, we can start to deduce that this issue is related to the way subject and issuer certificates are checked in the handling of jar (or, more precisely, apk) files in Android. Again, at line 10, we see an argument has been added to findCert; subjectCert. 

The final change in this patch is as follows:

{% highlight java %}
1 -    private static X509Certificate findCert(Principal issuer, X509Certificate[] candidates) {
2 +    private static X509Certificate findCert(Principal issuer, X509Certificate[] candidates,
3 +            X509Certificate subjectCert, boolean chainCheck) {
4          for (int i = 0; i < candidates.length; i++) {
5              if (issuer.equals(candidates[i].getSubjectDN())) {
6 +                if (chainCheck) {
7 +                    try {
8 +                        subjectCert.verify(candidates[i].getPublicKey());
9 +                    } catch (Exception e) {
10 +                        continue;
11 +                    }
12 +                }
13                  return candidates[i];
14              }
15          }
{% endhighlight %}

As expected, Line 3 shows the extra paramaters added to the findCert method. Finally, we see where chainCheck is used; line 6 checks if chainCheck is true, and if yes, verifies the public key of the issues (line 5) against the subject cert.

After assessing the patch, we understand that the issue described as Fake ID arises because there is no verification that the signing certificates private key is signed by the issuers public key. This means that we can create a malicious application and add another application certificate to the certificate chain. If at any point, our malicious application is loaded and the signatures of the certificates embedded in the application are used to decide on whether our application should be granted access to a resource, we may be able to bypass that check. The example given by Bluebox is related to an Adobe plugin loading in a webview, but it may be possible to bypass the Android permission model and access another apps sandbox. 

##Adobe and WebView

If we look at the PluginManager source code (https://android.googlesource.com/platform/frameworks/base/+/e3f9071/core/java/android/webkit/PluginManager.java) we see the following code which checks if two signatures are equal:

{% highlight java %}
private static boolean containsPluginPermissionAndSignatures(PackageInfo pkgInfo) {
...
	for (Signature signature : signatures) {
		for (int i = 0; i < SIGNATURES.length; i++) {
			if (SIGNATURES[i].equals(signature)) {
				signatureMatch = true;
{% endhighlight %}

Following the code back, we see that SIGNATURE contains only one signature (which we'll revist at later).

{% highlight java %}
private static final String SIGNATURE_1 = "308204c53082....";
...
private static final Signature[] SIGNATURES = new Signature[] {
	new Signature(SIGNATURE_1)
};
{% endhighlight %}

The other signature in the comparison is a list of all signatures (or certificates) in the package:

{% highlight java %}
Signature signatures[] = pkgInfo.signatures;
{% endhighlight %}

From our analysis of the patch for Google bug 13678484, we know in certain situations it is possible to add other certificates or signatures to a package, thereby passing this check. The next logical question is, what is the hardcoded signature? The first byte, 0x30, gives us an indication we're dealing with a DER encoded certificate. With some python and keytool, we can find out what the embedded certificate is (the signature has been shorted for readability).


    $ python -c "print('308204c5308203ada003020102020900d7cb412f7...'.decode('hex'))"| keytool -printcert
    Owner: CN=Adobe Systems Incorporated, OU=Information Systems, O=Adobe Systems Incorporated, L=San Jose, ST=California, C=US
    Issuer: CN=Adobe Systems Incorporated, OU=Information Systems, O=Adobe Systems Incorporated, L=San Jose, ST=California, C=US
    Serial number: d7cb412f75f4887e
    Valid from: Thu Oct 01 01:23:14 BST 2009 until: Mon Feb 16 00:23:14 GMT 2037
    Certificate fingerprints:
    	     MD5:  C0:64:76:66:D2:22:02:7E:94:29:BA:B1:51:67:65:A3
             SHA1: D5:CA:82:CB:DD:D3:98:52:79:B0:74:EA:53:BA:9A:30:68:89:35:BE
	     SHA256: 40:C1:2D:4A:8F:7D:95:55:A3:03:D5:FA:85:2C:C5:07:A4:75:46:53:66:07:2B:15:2C:FE:D1:3A:FF:AC:50:F3
	     Signature algorithm name: SHA1withRSA
	     Version: 3

    Extensions: 

    #1: ObjectId: 2.5.29.35 Criticality=false
    AuthorityKeyIdentifier [
    KeyIdentifier [
    0000: 5A F4 18 E4 19 A6 39 E1   65 7D B9 60 99 63 64 A3  Z.....9.e..`.cd.
    0010: 7E F2 0D 40                                        ...@
    ]
    [CN=Adobe Systems Incorporated, OU=Information Systems, O=Adobe Systems Incorporated, L=San Jose, ST=California, C=US]
    SerialNumber: [    d7cb412f 75f4887e]
    ]

    #2: ObjectId: 2.5.29.19 Criticality=false
    BasicConstraints:[
      CA:true
      PathLen:2147483647
    ]

    #3: ObjectId: 2.5.29.14 Criticality=false
    SubjectKeyIdentifier [
    KeyIdentifier [
    0000: 5A F4 18 E4 19 A6 39 E1   65 7D B9 60 99 63 64 A3  Z.....9.e..`.cd.
    0010: 7E F2 0D 40                                        ...@
    ]
    ]

This is the Adobe CA certificate. By reading the PluginManager source code, it becomes clear that the PluginManager class looks for browser plugins installed on the device, and decides to load them based on the inclusion of a Adobe certificate. This is due to the assumption that a package will only contain certificates validated on install, which we can see from Fake ID that this is not necessarily the case. The loading of browser plugins signed by Adobe happens automatically in all WebViews if the WebView has plugins enabled.

##How bad is this?
These two bugs (although now fixed) could be used to gain access to another application's sandbox and execute code in the context of that application, but only if an application uses WebViews and with plugins enabled. The malicious application would be able to read/write files belonging to another application and perform actions with the privileges of the application. To exploit this, however, an attacker would need to get malware on a device so naturally, the risk is lowered.

It is important to remember, though, that the WebView example is just *one* possible attack vector. The slightly scarier conclusion is that depending on where this same method of signature checking is performed, it may be possible to trick Android to loading an application with the signature of another, breaking any assumptions that may be in place. 

The next stage of this research will be to create a Proof of Concept for Fake ID and search for other places in the Android source code where signature checking is limited to checking whether a signature is contained within PackageInfo.
