# Salla + Stape Server-Side Tracking Skill

[![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey.svg)](LICENSE)

A [Claude Skill](https://support.claude.com/en/articles/12512180-use-skills-in-claude) for building, auditing and debugging server-side tracking on **Salla** stores using **Stape**-hosted server GTM.

Maintained by [Mitrone](https://mitrone.com).

[العربية](#العربية)

## Why use it

Ad platforms optimise on the conversions they receive. On Salla stores, browser-only pixels miss a large share of purchases, Salla's native integrations often double-count, and guest or Apple Pay orders frequently reach the platforms without customer data. This skill gives Claude the procedures and checks needed to fix that:

- Send complete, deduplicated conversions to GA4, Google Ads, Meta, TikTok and Snapchat
- Improve match quality by passing correct customer data, including for guest checkouts
- Identify whether a diagnostic comes from your setup or from Salla's legacy integrations before changing anything
- Get step-by-step, verifiable instructions instead of generic advice

## What's included

- Stape Conversion Tracking app for Salla: custom loader, Cookie Keeper, Stape User ID, webhooks
- Salla dataLayer reference: which events to use, which to ignore, and known data issues with fixes
- Server-side configuration for GA4, Google Ads, Meta CAPI, TikTok Events API, Snapchat CAPI and Mixpanel
- Browser and server deduplication rules per platform
- Webhook-based enrichment of guest purchases using Stape Data Store, with template code
- Net purchase value (excluding VAT and shipping), with template code
- Detecting and removing Salla's native pixels and conversion API apps
- Stape logs, quota and a debugging checklist

## Installation

### Claude.ai (web and desktop)

1. Download `salla-stape-tracking.zip` from the [latest release](../../releases/latest). Do not unzip it.
2. Go to **Settings → Capabilities** and enable **Code execution and file creation**.
3. Go to **Customize → Skills**, click **+**, then **Create skill → Upload a skill**, and select the ZIP file.
4. Make sure the skill is toggled on.

### Claude Code

Skills enabled on your claude.ai account sync automatically when you sign in with the same account. Run `/skills` to confirm.

To install manually, copy the `salla-stape-tracking` folder into `~/.claude/skills/`.

## Usage

Ask Claude in plain language. The skill loads automatically when the request is about Salla, Stape or server-side tracking. Examples:

```
Audit the server-side tracking on my Salla store.
Why does Meta report more purchases than my store?
Set up Snapchat CAPI through Stape with deduplication.
Guest and Apple Pay orders reach Meta without customer data. How do I fix it?
```

## Repository structure

```
salla-stape-tracking/
└── SKILL.md
```

## Disclaimer

Salla, Stape and the ad platforms change their interfaces and APIs regularly. The skill is based on live implementations as of October 2026. Review every change and test it in GTM Preview before publishing.

## License

Licensed under [CC BY-NC 4.0](LICENSE).

- Free for store owners to use on their own stores, and for personal and educational use, with attribution to Mitrone.
- Commercial use requires permission. This includes selling the skill, including it in paid products or courses, rebranding it, or using it to deliver paid services to clients.

For a commercial license, contact Mitrone on [WhatsApp](https://wa.me/201121150270).

## Professional services

Mitrone implements and audits server-side tracking for Salla and other e-commerce platforms.

- Website: [mitrone.com](https://mitrone.com)
- WhatsApp: [+20 112 115 0270](https://wa.me/201121150270)

---

<div dir="rtl">

## العربية

**Skill لـ Claude** تخليه يساعدك تبني وتراجع وتصلح التتبع من السيرفر لمتاجر **سلة** باستخدام **Stape**.

### الفايدة

- منصات الإعلانات تستلم كل الطلبات من غير تكرار، فأرقامك تبقى دقيقة والحملات تتعلم صح
- بيانات العميل توصل للمنصات حتى في طلبات الضيف و Apple Pay
- تعرف المشكلة جاية من إعداداتك ولا من تكاملات سلة القديمة قبل ما تغير أي حاجة
- تعليمات خطوة بخطوة تقدر تتأكد منها

### التثبيت على claude.ai

1. نزّل ملف `salla-stape-tracking.zip` من [آخر إصدار](../../releases/latest) من غير ما تفكه
2. من **Settings ← Capabilities** فعّل **Code execution and file creation**
3. من **Customize ← Skills** دوس **+** ثم **Create skill ← Upload a skill** واختار الملف
4. اتأكد إن الـ skill متفعلة

### الاستخدام

اسأل Claude عادي بالعربي، مثلا: "راجع التتبع في متجري على سلة" أو "ليه ميتا بتحسب الطلبات أكتر من المتجر؟"

### الترخيص

مجانية لأصحاب المتاجر في متاجرهم وللاستخدام الشخصي والتعليمي مع ذكر ميترون. البيع أو الدمج في منتجات أو كورسات مدفوعة أو تغيير الاسم أو استخدامها في تقديم خدمات مدفوعة للعملاء يحتاج إذن من ميترون. للتواصل: [واتساب](https://wa.me/201121150270) · [mitrone.com](https://mitrone.com)

</div>
