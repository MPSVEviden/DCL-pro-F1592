# DCL-pro-F1592 – Aktualizovaná dokumentace

# Komplexní shrnutí řešení autorizace KFZKZ pro Fiori Asset Worklist (upraveno dle testování a diskuse)

Tento dokument shrnuje **kompletní návrh, implementaci, úpravy a poznatky z testování** autorizace podle hodnoty **KFZKZ** (v datovém modelu jako *VehicleLicensePlateNumber*) u Fiori aplikace **Asset Master Worklist (F1592)**.

---

# 1. Důvod úpravy

Standardní aplikace F1592 neumožňuje filtrovat majetek podle objektu/budovy (KFZKZ).  
Zákazník požaduje:

- Každý uživatel smí vidět **pouze majetek v objektech**, na které má oprávnění.
- Uživatel může mít přiřazeno více objektů.
- Hodnoty objektů jsou v číselníku `ZAM_OBJEKT`.
- Hodnota KFZKZ je v datovém modelu dostupná jako **VehicleLicensePlateNumber**.
- Řádky s prázdným objektem **nemají být vidět**, pokud uživatel **nemá oprávnění i na prázdnou hodnotu** (změna oproti původní verzi).
- Chování **k datu výkazu (Key Date)** je plně pokryto standardním modelem ANLZ (adatu/bdatu).

---

# 2. Jak řešení funguje

Řešení je založené na kombinaci:

1. **Autorizační pole** `ZKFZKZ`
2. **Autorizační objekt** `Z_OBJ_KFZ`
3. **PFCG role** (např. `ZAM_FAA_KFZKZ_10000001`)
4. **Z‑DCL přístupová role** `ZR_FAA_KFZKZ`

DCL se vyhodnocuje **společně** se standardním SAP DCL → výsledkem je průnik (AND).

To znamená:
- Standardní oprávnění (např. ANLKL, KOSTL, GSBER) zůstávají platná.
- Naše Z‑DCL přidává další filtr – **KFZKZ**.

---

# 3. Tvorba autorizačního pole `ZKFZKZ` (SU20)

Pole je potřeba, aby jej bylo možné používat v SU21 a PFCG.

## 3.1 SE11 – využití existující domény `AM_KFZKZ`

- Typ: `CHAR`
- Délka: 15
- Value Table: `ZAM_OBJEKT`

## 3.2 SE11 – vytvoření Data Elementu

- Název: `AM_KFZKZZ`
- Doména: `AM_KFZKZ`
- Popisky: „Objekt“, „KFZKZ“

## 3.3 SU20 – creation of authorization field

- Název: **`ZKFZKZ`**
- Data element: `AM_KFZKZ`
- Organizational level: **NE**

---

# 4. Autorizační objekt `Z_OBJ_KFZ` (SU21)

Umožňuje určovat přístup uživatele ke konkrétním objektům.

- Název: **`Z_OBJ_KFZ`**
- Pole:
  - `ACTVT`
  - `ZKFZKZ`
- Použití: v DCL přes `aspect pfcg_auth()`

---

# 5. Role `ZAM_FAA_KFZKZ_10000001` (PFCG)

Role reprezentuje přístup na jeden konkrétní objekt.

Typické vyplnění:
- Objekt: `Z_OBJ_KFZ`
  - `ACTVT` = 03
  - `ZKFZKZ` = 10000001
- Role se přiřadí uživateli.
- Je možné založit více rolí – pro každý objekt jednu.

---

# 6. CDS Access Control `ZR_FAA_KFZKZ` – AKTUALIZOVANÁ VERZE

Na základě diskuze s business:
- **prázdné KFZKZ se NEMAJÍ zobrazovat** (původně se zobrazovat měly),
- zobrazí se pouze tehdy, **pokud má role povoleno i KFZKZ = ''**.

### Platná verze DCL:

```abap
@EndUserText.label: 'KFZKZ filter for C_FixedAssetWorklist'
@MappingRole: true
define role ZR_FAA_KFZKZ {

  grant select on C_FixedAssetWorklist
    where
      // uživatel musí mít výslovné oprávnění na hodnotu VehicleLicensePlateNumber
      ( VehicleLicensePlateNumber ) =
           aspect pfcg_auth( Z_OBJ_KFZ, ZKFZKZ, ACTVT = '03' );
}
```

### Změna:
- Varianta „prázdné vidí všichni“ byla odstraněna.
- Pokud je hodnota prázdná → zobrazí se pouze tehdy, pokud PFCG role **explicitně** obsahuje prázdnou hodnotu.

---

# 7. Chování na DEV/DF2 (poznatek z testování)

Na vývojových systémech DF2/DEV běžně platí:

- Vývojáři mají typicky **široká oprávnění**, často i nepřímo:
  - SAP_ALL
  - S_DEVELOP
  - super-role pro FE integraci
  - fallback roles z template
- I když není explicitně přiřazena role Z_OBJ_KFZ, může být oprávnění splněno přes:
  - široké FI-AA role,
  - generické AM/FI role,
  - fallback objekt ANLKL/BUKRS, přes který projdou i další kontroly.

Proto se může stát, že vývojář **vidí všechny záznamy**, i když nemá „viditelné“ Z‑role.

**Doporučení:**  
Test vždy provádět přes **testovací účet** bez developerských rolí.

---

# 8. Chování k datu výkazu (Key Date)

Nyní potvrzeno:

- KFZKZ je uloženo v tabulce **ANLZ** včetně časové platnosti `ADATU` a `BDATU`.
- CDS `C_FixedAssetWorklist` využívá parametr **P_KeyDate**.
- Tím se automaticky filtruje:
  - platnost záznamů,
  - historická data,
  - změny KFZKZ v čase.

**DCL časovou platnost neřeší**, řeší ji datový model.

---

# 9. „Záhadné záznamy navíc“ (DU5)

- Na DU5 se objevuje 8 záznamů navíc.
- Data na DU5 jsou neúplná / nečistá.
- Analýza se provede až na DU4 (kopie produkce).
- Vedeno jako **TODO**.

---

# 10. End‑to‑end tok

1. Fiori předá P_KeyDate.
2. CDS vyfiltruje majetek podle ANLZ k danému dni.
3. Standardní DCL aplikuje FI-AA oprávnění.
4. Z‑DCL aplikuje filtr podle KFZKZ.
5. Pokud KFZKZ není v roli → záznam se skryje.
6. Pokud je KFZKZ prázdné → zobrazí se jen při oprávnění na prázdnou hodnotu.

---

# 11. Shrnutí pro business

- Uživatel vidí **pouze majetek v objektech**, na které má roli.
- Prázdné KFZKZ se nezobrazují.
- Historie je podporována automaticky (ANLZ).
- Řešení je čisté, udržovatelné a bezpečné.
- Test je nutné provádět na ne‑developerských účtech.

---

# 12. Závěr

Řešení je plně funkční a sladěné s požadavky zákazníka.  
Dalším krokem je ověření na DU4 nad produkčními daty.
