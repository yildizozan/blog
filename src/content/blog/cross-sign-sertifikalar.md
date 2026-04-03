---
title: "Cross-Sign Sertifikalar: Belki de İhtiyacınız Var Ancak Haberiniz Yok"
description: "TLS sertifika zincirinin derinliklerine iniyoruz. Cross-sign sertifikaların ne olduğunu, neden gerektiğini ve erişim sorunlarını nasıl çözdüğünü açıklıyoruz."
pubDate: "Apr 04 2026"
---

Yine TLS yenileme zamanı geldi. Gittiniz, satın aldınız veya Let's Encrypt ya da Cloudflare ile ücretsiz edindiniz, manuel veya otomatik şekilde yenilediniz ve genelde de sorunsuz atlattınız. Belirli bir süre sonra erişim sorunları gelmeye başlarsa hemen aklımıza bir yerde yanlış yaptığımız gelir. Fakat sorun bundan çok daha derin olabilir.

> Not: Artık SSL yerine TLS kullanmamız gerekiyor. SSL, 1990'larda kullanılan ve güvenlik açıkları nedeniyle çoktan terk edilmiş eski bir protokol. Günümüzde TLS 1.2 ve TLS 1.3 kullanılıyor.

## Sertifikalar

Bir TLS sertifikası satın alındığında üç kısımdan oluşur:

- **Sunucu sertifikası** — Sitenize özgü, alan adınızı doğrulayan sertifika
- **Ara sertifika(lar)** — Kök ile sunucu sertifikası arasındaki köprü
- **Kök sertifika** — Tarayıcı ve işletim sistemlerinin güvendiği temel sertifika (CA - Certificate Authority tarafından yayımlanır)

Bu bir zincir şeklinde ilerler:

```
Kök Sertifika → Ara Sertifika 1 → Ara Sertifika 2 → ... → Sunucu Sertifikası
```

Her halka bir öncekini imzalar; böylece zincir doğrulanır. Tarayıcı, sunucu sertifikasından başlayarak zinciri geriye doğru takip eder ve güvendiği bir kök sertifikaya ulaşırsa bağlantıyı güvenli kabul eder.

## Sectigo Örneği

Geçen sene aldığımız sertifikaya baktığımızda zincir şöyle görünürdü:

```
USERTrust RSA Certification Authority  (Kök)
    └── Sectigo RSA Domain Validation CA  (Ara)
            └── *.example.com  (Sunucu)
```

Görünürde her şey yolunda. Tarayıcılar bu zinciri tanıyor, yeşil kilit ikonunu görüyorsunuz. Peki ya bazı kullanıcılar "Bağlantınız güvenli değil" uyarısıyla karşılaşmaya başlarsa?

## Cross-Sign Sertifika Nedir?

Cross-sign (çapraz imzalı) sertifika, bir kök sertifikanın başka bir kök sertifika tarafından da imzalanmasıdır.

Bunu şöyle düşünün: Yeni kurulan bir CA'nın (Sertifika Otoritesi) kök sertifikası henüz tüm cihazların trust store'unda (güvenilen sertifikalar listesi) bulunmayabilir. Özellikle eski Android cihazlar, eski Java sürümleri veya güncellenmemiş tarayıcılar bu yeni kök sertifikayı tanımaz.

Çözüm: Yeni kök sertifikayı, herkesin zaten güvendiği eski ve köklü bir CA imzalar. Böylece eski cihazlar bile yeni CA'nın sertifikalarına güvenebilir — çünkü onların güvendiği eski kök, yeni kökü onaylamıştır.

## Neden Gereklidir?

En çarpıcı gerçek dünya örneği **Let's Encrypt**'in 2021 yılında yaşadığı krizdir.

Let's Encrypt, kendi kök sertifikası olan **ISRG Root X1**'i kullanmaya başlamadan önce, köklü CA **IdenTrust**'ın **DST Root CA X3** sertifikasıyla cross-sign anlaşması yapmıştı. Bu sayede:

- Eski Android cihazlar (Android 7.1 öncesi) ISRG Root X1'i tanımasa bile DST Root CA X3 üzerinden Let's Encrypt sertifikalarına güvenebiliyordu.
- Milyonlarca site sorunsuz çalışıyordu.

**30 Eylül 2021'de** DST Root CA X3'ün geçerlilik süresi doldu.

Android 7.1 ve altı sürümleri çalıştıran cihazlarda Let's Encrypt sertifikası kullanan sitelere erişim kesildi. Bu, dünya genelinde milyonlarca eski cihazı etkileyen büyük bir kesintiye yol açtı.

Özetle: Sertifikanız tamamen geçerliydi, hiçbir yanlışlık yapmamıştınız — ama kullanıcılarınızın bir kısmı sitenize erişemiyordu.

## Sorun Nasıl Tespit Edilir?

Sertifika zincirinizi kontrol etmek için `openssl` kullanabilirsiniz:

```bash
openssl s_client -connect example.com:443 -showcerts
```

Çıktıda kaç tane sertifika gördüğünüze dikkat edin. Sadece sunucu sertifikasını görüyorsanız, ara sertifikalar sunucuda eksik demektir.

Alternatif olarak [SSL Labs](https://www.ssllabs.com/ssltest/) gibi online araçlar zinciri görsel olarak gösterir ve eksik halkaları işaretler.

## Nasıl Çözülür?

**1. Ara sertifikaları eksiksiz yapılandırın**

Sunucunuza sertifika yüklerken genellikle iki dosya gelir: `certificate.crt` ve `ca-bundle.crt`. Ca-bundle dosyası ara sertifikaları içerir ve sunucu yapılandırmanıza mutlaka eklenmelidir.

Nginx için:
```nginx
ssl_certificate     /path/to/certificate.crt;
ssl_certificate_key /path/to/private.key;
# Ara sertifikalar certificate.crt dosyasına zaten dahil edilmeli
# Ya da ayrı bir chain dosyası kullanılabilir
```

**2. Cross-sign destekli zincir kullanın**

Eğer CA'nız cross-sign sertifika sunuyorsa (Let's Encrypt bunu varsayılan olarak yapıyor), zincir dosyasında hem yeni hem de eski kök üzerinden giden yol bulunur. Böylece hem yeni hem de eski istemciler doğru zinciri bulabilir.

**3. Eski istemci desteğini değerlendirin**

Kullanıcı kitlenizde çok sayıda eski Android ya da Java tabanlı sistem varsa, CA seçiminizde cross-sign desteğine dikkat edin veya `fullchain.pem` gibi tüm zinciri içeren dosyaları kullandığınızdan emin olun.

## Sonuç

TLS sertifika sorunları her zaman yanlış yaptığınız bir şeyden kaynaklanmaz. Bazen güvendiğiniz altyapının altındaki bir katmanda, farkında olmadığınız bağımlılıklar vardır.

Cross-sign sertifikaları bu bağımlılıkların en sık karşılaşılan örneğidir. Sertifikanızı doğru aldınız, doğru kurduğunuz, ama birinin güven zinciri sessizce sona erdi ve bunu ancak kullanıcı şikayetleriyle öğrendiniz.

Bir sonraki TLS yenilemenizde sertifika zincirinin tamamını kontrol edin. Sadece "yeşil kilit var mı?" diye sormak yetmez; "hangi istemciler bu zinciri doğrulayabilir?" sorusunu da sormak gerekir.
