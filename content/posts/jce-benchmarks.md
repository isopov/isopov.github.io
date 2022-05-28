+++ 
draft = false
date = 2012-05-18T20:44:46+03:00
title = "Java Cryptography Extension Providers Performance"
tags = ["Java"]
+++

The [Java Cryptography Extensions](https://en.wikipedia.org/wiki/Java_Cryptography_Extension) exist for many years, They allow to use cryptography that is not bundled in the Java platform itself. I've met [Bouncycastle](https://www.bouncycastle.org/) crypto provider library many times in projects I worked on. But there are other third party crypto providers as well. At least two of them - [Consrypt](https://github.com/google/conscrypt) and [Corretto](https://github.com/corretto/amazon-corretto-crypto-provider) (named the same way as [Openjdk distribution by Amazon](https://aws.amazon.com/corretto)) declare high performance as their features. I decided to measure these claims and wrote some [benchmarks](https://github.com/isopov/jce-benchmarks). To not compare apples with oranges I needed some algorithms that were implemented in all different crypto providers that I tried to test. So the choice may seem a bit strange.

First goes benchmark for Cipher doing AESGCMNoPadding decrypting and encrypting. All four provider can do it with bundled Java 17 provider being the most performant.

{{< chart 90 200 >}}
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
    }
}
{{< /chart >}}

TODO: other benches

{{< chart 90 200 >}}
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
    }
}
{{< /chart >}}
