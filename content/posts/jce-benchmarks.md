+++ 
draft = false
date = 2022-05-28T20:44:46+03:00
title = "Java Cryptography Extension Providers Performance"
tags = ["Java"]
+++

The [Java Cryptography Extensions](https://en.wikipedia.org/wiki/Java_Cryptography_Extension) exist for many years, They allow to use cryptography that is not bundled in the Java platform itself. I've met [Bouncycastle](https://www.bouncycastle.org/) crypto provider library many times in projects I worked on. But there are other third party crypto providers as well. At least two of them - [Consrypt](https://github.com/google/conscrypt) and [Corretto](https://github.com/corretto/amazon-corretto-crypto-provider) (named the same way as [Openjdk distribution by Amazon](https://aws.amazon.com/corretto)) declare high performance as their features. I decided to measure these claims and wrote some [benchmarks](https://github.com/isopov/jce-benchmarks). To not compare apples with oranges I needed some algorithms that were implemented in all different crypto providers that I tried to test. So the choice may seem a bit strange.

First goes benchmark for Cipher doing AESGCMNoPadding (and that is the only cipher supported in all 4 provider that I have found) decrypting and encrypting (measured in operations per second - higher is better). All four providers can do it with bundled Java 17 provider being the most performant.

{{< chart 90 300 >}}
{
    type: 'bar',
    data: {
        labels: ['Decrypt', 'Encrypt'],
        datasets: [
          {
              label: 'Default',
              data: [2741608, 1810941],
              backgroundColor: 'yellow',
            },
            {
              label: 'Corretto',
              data: [1522272, 1560521],
              backgroundColor: 'red',
            },
            {
              label: 'Bouncycastle',
              data: [1355433, 423344],
              backgroundColor: 'green',
            },
            {
              label: 'Conscrypt',
              data: [1330481, 1103269],
              backgroundColor: 'blue',
            }
          ],
    },
    options: {
        maintainAspectRatio: false,
        scales: {
            yAxes: [{
                ticks: {
                    beginAtZero: true
                }
            }]
        }
    }
}
{{< /chart >}}

Here almost up-to-date JVM (Java 18 is already released) shines. But lets see the performance of [hash-based message authenication codes](https://en.wikipedia.org/wiki/HMAC).

{{< chart 90 300 >}}
{
    type: 'bar',
    data: {
        labels: ['md5', 'sha1', 'sha256', 'sha512'],
        datasets: [
          {
              label: 'Default',
              data: [1217450, 793005, 807643, 642063],
              backgroundColor: 'yellow',
            },
            {
              label: 'Corretto',
              data: [1221880, 1287582, 815088, 575937],
              backgroundColor: 'red',
            },
            {
              label: 'Bouncycastle',
              data: [1123098, 814171, 588554, 386837],
              backgroundColor: 'green',
            },
            {
              label: 'Conscrypt',
              data: [318263, 321959, 258560, 179285],
              backgroundColor: 'blue',
            }
          ],
    },
    options: {
        maintainAspectRatio: false,
        scales: {
            yAxes: [{
                ticks: {
                    beginAtZero: true
                }
            }]
        }
    }
}
{{< /chart >}}

Here Corretto outperforms all other options for sha1 hashing, but in other cases is is almost the same as default Openjdk crypto provider. Consrypt is real outsider here.

Now lets look at signing our messages with crypto signatures.

{{< chart 90 300 >}}
{
    type: 'bar',
    data: {
        labels: ['SHA1withECDSA', 'SHA256withECDSA', 'SHA512withECDSA', 'SHA1withRSA', 'SHA256withRSA', 'SHA512withRSA'],
        datasets: [
          {
              label: 'Default',
              data: [1117, 1110, 1124, 575, 568, 568],
              backgroundColor: 'yellow',
            },
            {
              label: 'Corretto',
              data: [798, 793, 789, 1221, 1229, 1233],
              backgroundColor: 'red',
            },
            {
              label: 'Bouncycastle',
              data: [329, 336, 325, 517, 518, 511],
              backgroundColor: 'green',
            },
            {
              label: 'Conscrypt',
              data: [34319, 34654, 34299, 1231, 1231, 1215],
              backgroundColor: 'blue',
            }
          ],
    },
    options: {
        maintainAspectRatio: false,
        scales: {
            yAxes: [{
                ticks: {
                    beginAtZero: true
                }
            }]
        }
    }
}
{{< /chart >}}

Conscrypt really shines here with [ECDSA](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm) implementations from [BoringSSL](https://boringssl.googlesource.com/boringssl/). With [RSA](https://en.wikipedia.org/wiki/RSA_(cryptosystem)) its performance is also among leaders. However it is interesting that bundled provider outperforms Corretto with ECDSA, but loses with RSA.

Now lets look at verifying those signatures.

{{< chart 90 300 >}}
{
    type: 'bar',
    data: {
        labels: ['SHA1withECDSA', 'SHA256withECDSA', 'SHA512withECDSA'],
        datasets: [
          {
              label: 'Default',
              data: [581, 587, 578],
              backgroundColor: 'yellow',
            },
            {
              label: 'Corretto',
              data: [940, 939, 944],
              backgroundColor: 'red',
            },
            {
              label: 'Bouncycastle',
              data: [724, 724, 768],
              backgroundColor: 'green',
            },
            {
              label: 'Conscrypt',
              data: [12893, 12807, 12759],
              backgroundColor: 'blue',
            }
          ],
    },
    options: {
        maintainAspectRatio: false,
        scales: {
            yAxes: [{
                ticks: {
                    beginAtZero: true
                }
            }]
        }
    }
}
{{< /chart >}}

Once again for ECDSA verifying Conscrypt outperforms everyone else. For RSA numbers are so different for all options, lets place them on separate chart.

{{< chart 90 300 >}}
{
    type: 'bar',
    data: {
        labels: ['SHA1withRSA', 'SHA256withRSA', 'SHA512withRSA'],
        datasets: [
          {
              label: 'Default',
              data: [16684, 16616, 16688],
              backgroundColor: 'yellow',
            },
            {
              label: 'Corretto',
              data: [38029, 37155, 37855],
              backgroundColor: 'red',
            },
            {
              label: 'Bouncycastle',
              data: [16928, 16712, 17170],
              backgroundColor: 'green',
            },
            {
              label: 'Conscrypt',
              data: [40487, 40505, 39226],
              backgroundColor: 'blue',
            }
          ],
    },
    options: {
        maintainAspectRatio: false,
        scales: {
            yAxes: [{
                ticks: {
                    beginAtZero: true
                }
            }]
        }
    }
}
{{< /chart >}}

Here both Corretto and Consrypt do something to prove their self-names of high-performance implementations.

What should you use in the end? I don't think that performance is actually what you should look at in the first place. You should start with security and features that you need for your application. That is why Bouncycastle provider whose performance is not that great can be met so often.

