# SUQ — a point of sale built for Qatar

**[Live demo →](https://yourdomain.com)**

Two working prototypes and a product blueprint for a restaurant and supermarket platform designed
around Qatar's actual regulatory and commercial conditions, rather than a Gulf-generic system with
the currency swapped.

Built by Irsh — Doha, Qatar.

---

## The problem this is solving

Every point of sale sold in Doha is a Dubai or Riyadh product with `AED` or `SAR` replaced by `QAR`.
That is not localisation, because the differences that matter are structural:

- **Qatar has no VAT.** It signed the GCC framework agreement but has never enacted the law. A system
  that hardcodes a 5% rate is wrong today; a system with no tax engine at all is wrong the day the
  law lands.
- **Qatar does have excise** — 100% on tobacco and energy drinks, 50% on carbonated drinks — and it
  is levied at import, not at the till. It belongs in landed cost and margin. Imported systems either
  add it as a receipt line (wrong, and a compliance problem) or ignore it (wrong, and a margin problem).
- **The payment rails are national.** NAPS and Himyan, QPay, QMP, and Fawran, which at a flat fee of
  roughly half a riyal is the cheapest tender in the country and is invisible in most tills.
- **Wages run through WPS**, and staff documents — QID, work permit, Baladiya health card — gate who
  can legally work a shift.
- **The calendar is different.** Friday–Saturday weekend, five daily prayer breaks, and Ramadan, which
  moves about eleven days earlier each Gregorian year. Year-on-year comparatives have to run on the
  Hijri calendar or they are wrong by construction.

## What's in here

| | |
|---|---|
| **[The till](public/pos.html)** | Restaurant and supermarket editions on one core. Full RTL Arabic, dormant VAT engine, Qatari tender rails with live cost-per-tender, cash rounding to 0.25 QAR, carrier-bag levy, prayer countdown with service hold, and a compliance dock covering staff documents, cold chain, and WPS readiness. |
| **[Landed cost & inventory](public/inventory.html)** | An editable shipment cost engine. Apportion freight by value, weight, or unit count and watch unit costs move. Duty on CIF, excise capitalised into stock, VAT-day margin compression modelling, a goods-receipt gate on remaining shelf life, and stock cover run against route lead times with Hijri-anchored demand factors. |
| **[Product blueprint](public/blueprint.md)** | Module map, data model, phased roadmap, commercial model, and an explicit list of the regulatory questions that must be verified with the General Tax Authority before any of it ships. |

## Engineering decisions worth noting

**Tax is versioned configuration, never a constant.** Rates live in a `tax_rate` table with
`effective_from` and a `legal_reference`. Today every row is `0.00`. VAT day is one insert per tax
group, not a deployment.

**Every sale is a structured document from the start.** Qatar's Cabinet approved a draft e-invoicing
law in May 2026, with a Peppol-style architecture widely expected. A till that stores receipt text
instead of a structured invoice is a rewrite when that lands, so the invoice model carries gapless
sequencing, a buyer TRN slot, a QR payload slot, and a five-year archive from day one.

**Arabic is a peer field, not a translation table.** `name_ar` sits alongside `name_en` on the product
record and is required. Translation-layer approaches break the moment a product has no English name,
which in this market is common.

**Freight apportionment basis is exposed to the user.** Apportioning a mixed container by value gives
almost all the freight to the high-value lines; by weight it goes to the heavy ones. Most systems
hardcode one basis and quietly misprice half the range. This one makes it a decision.

## Running it locally

Everything is static — no build step, no dependencies, no backend.

```bash
npx serve public
# or: python3 -m http.server -d public 8000
```

## Deploying

Pushing to `main` triggers `.github/workflows/deploy.yml`, which publishes `public/` to GitHub Pages.
For a custom domain, rename `public/CNAME.example` to `public/CNAME` and put your domain in it, then
point the domain's DNS at GitHub Pages.

## Status and disclaimer

Prototypes with sample data, built to demonstrate the domain model and the interaction design.
Rates and figures are illustrative and are **not** tax, customs, legal, or pricing advice. The tax
position reflects Qatar as at August 2026 and must be re-verified with the General Tax Authority,
the Ministry of Commerce and Industry, and a licensed clearing agent before any production use.

## License

MIT — see [LICENSE](LICENSE).
