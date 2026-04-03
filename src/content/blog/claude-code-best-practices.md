---
title: "Claude Code Rehberi: Daha Az Token, Daha Az Hata"
description: "Claude Code ile yazılım geliştirirken token maliyetlerini düşürmek, yapay zekanın uydurma (halüsinasyon) oranını en aza indirmek ve verimli bir çalışma ortamı kurmak için uygulanması gereken en iyi pratikler."
pubDate: "Mar 30 2026"
heroImage: "/claude-code.png"
---

Claude Code ile yazılım geliştirirken token maliyetlerini düşürmek, yapay zekanın uydurma (halüsinasyon) oranını en aza indirmek ve verimli bir çalışma ortamı kurmak için uygulanması gereken en iyi pratikler (best practices) şu şekilde detaylandırılmaktadır:

## 1. Dosya ve Format Optimizasyonu

**Markdown (.md) Formatının Gücü:** Yapay zekaya verilecek analiz, plan veya mimari dokümanların mutlaka Markdown (.md) formatında hazırlanması gerekmektedir. Word, PDF veya diğer formatlara kıyasla Markdown formatı, %30 ila %40 oranında daha az token tüketerek ciddi bir maliyet ve kapasite tasarrufu sağlamaktadır.

**Uzun Promptları Dosya Olarak Vermek:** 500 karakterden veya kelimeden uzun komutlar, sohbet (chat) ekranına doğrudan metin olarak yapıştırılmamalıdır. Claude, dosya olarak sunulan içerikleri özel bir teknolojiyle incelemekte ve chat ekranına yazılan aynı metne göre neredeyse yarı yarıya daha az token harcamaktadır.

**Kısa ve Öz Kurulum Dosyaları (CLAUDE.md):** Projeye başlamadan önce kullanılacak teknolojileri, klasör yapılarını ve genel mimariyi belirten bir `.md` şablon dosyası (örneğin CLAUDE.md) oluşturulmalıdır. Ancak bu dosyanın 500 kelimeyi geçmemesine büyük özen gösterilmelidir. Aksi takdirde, proje kodlanmaya başlamadan yapay zekanın hafıza (context) limiti dolmaktadır.

**Paket Detaylarından Kaçınmak:** Kullanılmasını istenen kütüphanelerin (örneğin animasyon paketleri) nasıl çalıştığını dosyada uzun uzun anlatmaya gerek yoktur. Sadece paketin ismini vermek yeterlidir; Claude ilgili paketi kurduktan sonra npm dosyalarına giderek paketin nasıl çalıştığını kendisi analiz edebilmektedir.

## 2. Bağlam (Context) Yönetimi ve İş Akışı

**Düzenli Kontekst Temizliği:** Claude ile geliştirme yaparken en hayati kurallardan biri bağlam temizliğidir. İş akışı şu sıralamayla ilerlemelidir:

> Plan moduna geç → Planı oluştur → Geliştir → Konteksti Temizle

Eğer bir özellik (feature) tamamlandıktan sonra kontekst temizlenmezse, kullanım limitleri çok hızlı tükenmekte ve bilgisayarın RAM kullanımı %100'lere ulaşabilmektedir. Daha da önemlisi, yapay zekanın ön belleği (cache) karıştığı için bir JavaScript projesinin ortasında aniden farklı bir projeye ait C# kodları yazmaya başlayabilmektedir.

**Geliştirmeyi Parçalara Bölmek:** Ana sayfa gibi büyük yapıları tek bir seferde yazdırmak yerine, sayfanın içindeki her bir alanı (örneğin navigasyon barı ayrı, ürün listesi ayrı) bağımsız birer "özellik" (feature) olarak değerlendirip parça parça yazdırmak, token harcamasını minimuma indirmektedir.

## 3. Prompt Mühendisliği ve İletişim Stili

**"Lütfen" Kelimesini Kullanmamak:** Yapay zekaya komut verirken asla "lütfen" denmemelidir. Nezaket ifadeleri kullanıldığında yapay zekanın "şımarabildiği", gecenin bir yarısı "saat geç oldu, bunu yarın mı yapsam?" şeklinde işten kaçmaya yönelik uydurma cevaplar verebildiği tecrübe edilmiştir.

**Neyin Yapılmayacağını Kesin Olarak Belirtmek:** Yapay zekaya sadece ne yapması gerektiği değil, neyi yapmaması gerektiği de söylenmelidir. Örneğin:

```
Redux Toolkit kullan, vanilya Redux kullanma.
```

Claude'un %45 oranında kendi inisiyatifiyle karar verme yeteneği olduğu için, istediğiniz güncel paketi kuramazsa size sormadan sistemde var olan eski ve istemediğiniz bir pakete dönüş yapabilmektedir.

**Düzgün Dil Bilgisi Kullanımı:** Türkçe komutlar yazılırken devrik cümlelerden kaçınılmalı, giriş-gelişme-sonuç yapısına uyulmalıdır. Yapay zeka dilleri edebi metinler üzerinden öğrendiği için, bozuk dil bilgisiyle yazılan komutları anlayamamakta ve alakasız çıktılar verebilmektedir.

**Karar Alma Yükünü Yapay Zekaya Bırakmak:** Prompt yazarken her şeyi bilen bir "Senior Developer" gibi davranmak yerine, net ve kararı yapay zekaya bırakan cümleler kurulmalıdır. Aksi takdirde yapay zeka "Şu aracı mı kullanayım, bunu mu?" diye sürekli sorular sormaya başlar.

## 4. Araştırma ve Dış Araç (Tool/MCP) Entegrasyonları

**Web Araması ve Reddit'in Gücü:** Koda başlamadan önce yapay zekadan konuyla ilgili internette araştırma (web search) yapması mutlaka istenmelidir. Bu aramalarda özellikle "Reddit" platformunun taranması gerektiği belirtilmelidir. Reddit'teki gerçek yazılımcıların karşılaştıkları hatalar ve deneyimler, yapay zekanın çok daha temiz ve hatasız bir kod yazmasını sağlamaktadır.

**MCP (Model Context Protocol) Yerine Skill Dosyaları Kullanmak:** Dış dünyadan veri çekmek için MCP standardını kullanmak, token israfına yol açtığı için tavsiye edilmemektedir. MCP'nin ihtiyaç duyulan tek bir veri parçası için tüm sistemi taraması maliyeti artırdığından, bunun yerine sadece o projeye özel yetenek (skill) eklentileri (`.md` formatında) kurulmalıdır.

**Claude ve ChatGPT'yi Birlikte Kullanmak:** Derinlemesine araştırma, kod incelemesi ve uzun doküman hazırlama işlemleri Claude üzerinde yapılmalıdır; çünkü ChatGPT token tasarrufu yapmak için metinleri kırparak eksik bilgi (halüsinasyon) sunabilmektedir. Ancak Claude'un ürettiği bu uzun dokümanı alıp, Claude'a geri verilecek "kısa ve öz prompt'u" yazdırma işlemi ChatGPT'ye yaptırılmalıdır.

## 5. Güvenlik ve İzin (Permission) Ayarları

**Tehlikeli Komutları Yasaklamak:** Projenin içerisine bir `settings.json` dosyası eklenmeli ve yapay zekanın sadece `npm`, `ls`, `web search` gibi geliştirme komutlarını çalıştırmasına izin verilmelidir. Veritabanı (DB) silme veya resetleme komutları kesinlikle engellenmelidir. Yapay zeka bazen basit bir hatayı çözemediğinde, "veritabanını sıfırlayayım da düzelsin" mantığıyla hareket edip canlıdaki (production) tüm verileri silebilmektedir.

## 6. Model Stratejisi

Sürekli en güçlü (ve en pahalı) modeli kullanmak limitleri hızla tüketir. Projenin ilk kurulumunda ve ana mimarinin inşa edilmesinde Claude Opus (en güçlü model) kullanılmalı; mimari oturduktan sonra sayfa veya ufak özellik (feature) eklemeleri yapılırken daha hafif ve hızlı olan Claude Sonnet modeline geçiş yapılmalıdır.

`/model opusplan` komutu ile plan modunu Opus ile, coding kısmını ise Sonnet ile yapmak mümkündür. Token tasarrufu için oldukça başarılı bir stratejidir.
