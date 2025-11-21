[[_TOC_]]   

# V2

## DCL-pro-F1592 – Aktualizovaná dokumentace (včetně změny s REDEFINITION)

## Komplexní shrnutí řešení autorizace KFZKZ pro Fiori Asset Worklist
Tento dokument shrnuje **kompletní návrh, implementaci, změny, problémové chování a konečné řešení** autorizace podle hodnoty **KFZKZ** (v datovém modelu *VehicleLicensePlateNumber*) u Fiori aplikace **Asset Master Worklist (F1592)**.

---

## 1. Důvod úpravy

Standardní aplikace F1592 nedokáže filtrovat majetek podle objektu/budovy (KFZKZ).  
Zákazník však požaduje:

- Uživatel smí vidět **pouze majetek umístěný v objektech**, na které má oprávnění.  
- Uživatel může mít více objektů přiřazených.  
- KFZKZ pochází z číselníku `ZAM_OBJEKT`.  
- Prázdné hodnoty KFZKZ **nemají být vidět**, pokud nejsou výslovně povoleny.  
- Historický pohled přes Key Date je řešen již standardní logikou tabulky ANLZ (ADATU/BDATU).

---

## 2. Architektura řešení

Finální řešení se skládá z:

1. **Autorizační pole** `ZKFZKZ`  
2. **Autorizační objekt** `Z_OBJ_KFZ`  
3. **PFCG role** s povolenými objekty (např. `ZAM_FAA_KFZKZ_10000001`)  
4. **Z-DCL role** `ZR_FAA_KFZKZ`  
5. **Standardní SAP DCL** na `C_FixedAssetWorklist`

#### Klíčová zjištění
- Standardní a vlastní DCL se **kombinují logikou AND**, nikoliv OR.  
- Naše původní představa byla, že Z-DCL „ořezává“ výsledek standardního DCL.  
- Realita: pokud standardní DCL „pustí” záznam, dostane se do výstupu **ještě před KFZKZ**, a tím obejde náš filtr (pokud jsou splněna standardní oprávnění).

**To vysvětluje, proč uživatel viděl více záznamů (např. WERKS = 4800), i když KFZKZ ≠ 48000001.**

---

## 3. Finální řešení pomocí REDEFINITION

### 3.1 Problém
Standardní DCL poskytoval přístup na základě těchto objektů:

- A_S_ANLKL (třída majetku + společnost)  
- A_S_GSBER (obchodní úsek)  
- A_S_KOSTL (nákladové středisko)  
- A_S_WERK (závod)

Uživatel tak reálně splnil standardní DCL → a KFZKZ filtr se tím vůbec neaplikoval.

### 3.2 Řešení
Zavedli jsme:

#### 🔥 **REDEFINITION** DCL role

Pomocí klíčového slova:

```
REDEFINITION
```

se naše Z‑DCL stává **nahrazením** (override) standardního SAP DCL.  
To znamená:

- Standardní DCL se ignoruje  
- Pouze naše Z-DCL se vyhodnocuje  
- ALE → do Z-DCL jsme ručně překopírovali i standardní podmínky  
- Výsledek je kombinace „standardní logika AND KFZKZ filtr“

### 3.3 Výsledek
- Standardní přístupová logika FI‑AA zůstala zachována  
- Navíc se aplikuje náš **povinný filtr KFZKZ**  
- **Uživatel nyní vidí pouze majetek v konkrétním KFZKZ**  
- Záznamy pro WERKS 4800 již neprojdou, pokud KFZKZ ≠ 48000001  

---

## 4. Autorizační pole `ZKFZKZ` (SU20)

- Typ: CHAR 15  
- Doména: `AM_KFZKZ`  
- Data element: `AM_KFZKZZ`  
- Neorganizational level  
- Používá se v objektu SU21

---

## 5. Autorizační objekt `Z_OBJ_KFZ` (SU21)

| Pole | Význam |
|------|--------|
| ACTVT | 03 = Display |
| ZKFZKZ | Hodnota KFZKZ |

Objekt je volán přes `aspect pfcg_auth()` v DCL.

---

## 6. PFCG role

Příklad: `ZAM_FAA_KFZKZ_48000001`

- `Z_OBJ_KFZ`  
  - ACTVT = 03  
  - ZKFZKZ = 48000001  

Poznámka:  
Z derivované role (např. Z3AA91_00) se nepřenáší *hodnota*, pouze struktura.  
Hodnoty jsou definované až v konkrétní odvozené roli.

---

## 7. Finální Z‑DCL `ZR_FAA_KFZKZ` (s REDEFINITION)

```abap
@EndUserText.label: 'KFZKZ + Standard FI-AA Access Control for C_FixedAssetWorklist'
@MappingRole: true
define role ZR_FAA_KFZKZ
    REDEFINITION {

  grant select on C_FixedAssetWorklist
    where

      // 1) Standardní FI-AA logika (zkopírováno ze SAP standardu):

      ( AssetClass, CompanyCode ) =
          aspect pfcg_auth ( A_S_ANLKL, ANLKL, BUKRS, ACTVT = '03' )

      and ( ( CompanyCode, AssetBusinessArea ) =
          aspect pfcg_auth ( A_S_GSBER, BUKRS, GSBER )
            or AssetBusinessArea = '' )

      and ( ( CompanyCode, AssetCostCenter ) =
          aspect pfcg_auth ( A_S_KOSTL, BUKRS, KOSTL )
            or AssetCostCenter = '' )

      and ( ( CompanyCode, Plant ) =
          aspect pfcg_auth ( A_S_WERK, BUKRS, WERKS )
            or Plant = '' )

      // 2) Dodatečný filtr KFZKZ:

      and ( VehicleLicensePlateNumber ) =
            aspect pfcg_auth( Z_OBJ_KFZ, ZKFZKZ, ACTVT = '03' );

}
```

### Co se tím dosáhne:
- REDEFINITION „vypne“ standardní DCL  
- My ho „vracíme zpět“ ručním překopírováním  
- A zároveň doplňujeme náš filtr  
- Výsledkem je čisté chování typu:

```
(Standard FI-AA oprávnění) AND (KFZKZ)
```

---

## 8. Chování po úpravě na testu (validace)

#### Výchozí stav:
- 550 000 celkových záznamů
- Uživatel před úpravou viděl 37 000 (WERKS filtr)

#### Po úpravě:
- Už vidí **pouze majetek s KFZKZ = 48000001**
- Počet odpovídá skutečnosti (~7454 záznamů)
- Chování plně odpovídá očekávání businessu

---

## 9. Chování k historickým datům (Key Date)

Funguje kompletně standardně díky:
- ANLZ.ADTTU
- ANLZ.BDATU
- Propagaci parametru `P_KeyDate` z view

Žádná speciální logika v DCL není potřeba.

---

## 10. Shrnutí pro business

- Uživatel vidí majetek **pouze tam, kde má KFZKZ oprávnění**  
- Standardní FI-AA omezení se stále uplatňují  
- Prázdné KFZKZ se nezobrazují  
- Chování je čisté, auditovatelné, bezpečné  
- Historické řezy přes Key Date jsou podporovány automaticky  
- Chování na testu ověřeno (výsledně 7454 záznamů)

---

## 11. Závěr

Finalizované řešení využívá:

- vlastní autorizační objekt  
- vlastní DCL s REDEFINITION  
- ručně vložené standardní FI-AA podmínky  
- povinný filtr KFZKZ  

Výsledkem je přesné a stabilní řízení přístupu podle objektů (KFZKZ), plně v souladu s požadavky zákazníka.



# V1

## DCL-pro-F1592 – Aktualizovaná dokumentace

## Komplexní shrnutí řešení autorizace KFZKZ pro Fiori Asset Worklist (upraveno dle testování a diskuse)

Tento dokument shrnuje **kompletní návrh, implementaci, úpravy a poznatky z testování** autorizace podle hodnoty **KFZKZ** (v datovém modelu jako *VehicleLicensePlateNumber*) u Fiori aplikace **Asset Master Worklist (F1592)**.

---

## 1. Důvod úpravy

Standardní aplikace F1592 neumožňuje filtrovat majetek podle objektu/budovy (KFZKZ).  
Zákazník požaduje:

- Každý uživatel smí vidět **pouze majetek v objektech**, na které má oprávnění.
- Uživatel může mít přiřazeno více objektů.
- Hodnoty objektů jsou v číselníku `ZAM_OBJEKT`.
- Hodnota KFZKZ je v datovém modelu dostupná jako **VehicleLicensePlateNumber**.
- Řádky s prázdným objektem **nemají být vidět**, pokud uživatel **nemá oprávnění i na prázdnou hodnotu** (změna oproti původní verzi).
- Chování **k datu výkazu (Key Date)** je plně pokryto standardním modelem ANLZ (adatu/bdatu).

---

## 2. Jak řešení funguje

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

## 3. Tvorba autorizačního pole `ZKFZKZ` (SU20)

Pole je potřeba, aby jej bylo možné používat v SU21 a PFCG.

### 3.1 SE11 – využití existující domény `AM_KFZKZ`

- Typ: `CHAR`
- Délka: 15
- Value Table: `ZAM_OBJEKT`

### 3.2 SE11 – vytvoření Data Elementu

- Název: `AM_KFZKZZ`
- Doména: `AM_KFZKZ`
- Popisky: „Objekt“, „KFZKZ“

### 3.3 SU20 – creation of authorization field

- Název: **`ZKFZKZ`**
- Data element: `AM_KFZKZ`
- Organizational level: **NE**

---

## 4. Autorizační objekt `Z_OBJ_KFZ` (SU21)

Umožňuje určovat přístup uživatele ke konkrétním objektům.

- Název: **`Z_OBJ_KFZ`**
- Pole:
  - `ACTVT`
  - `ZKFZKZ`
- Použití: v DCL přes `aspect pfcg_auth()`

---

## 5. Role `ZAM_FAA_KFZKZ_10000001` (PFCG)

Role reprezentuje přístup na jeden konkrétní objekt.

Typické vyplnění:
- Objekt: `Z_OBJ_KFZ`
  - `ACTVT` = 03
  - `ZKFZKZ` = 10000001
- Role se přiřadí uživateli.
- Je možné založit více rolí – pro každý objekt jednu.

---

## 6. CDS Access Control `ZR_FAA_KFZKZ` – AKTUALIZOVANÁ VERZE

Na základě diskuze s business:
- **prázdné KFZKZ se NEMAJÍ zobrazovat** (původně se zobrazovat měly),
- zobrazí se pouze tehdy, **pokud má role povoleno i KFZKZ = ''**.

#### Platná verze DCL:

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

#### Změna:
- Varianta „prázdné vidí všichni“ byla odstraněna.
- Pokud je hodnota prázdná → zobrazí se pouze tehdy, pokud PFCG role **explicitně** obsahuje prázdnou hodnotu.

---

## 7. Chování na DEV/DF2 (poznatek z testování)

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

## 8. Chování k datu výkazu (Key Date)

Nyní potvrzeno:

- KFZKZ je uloženo v tabulce **ANLZ** včetně časové platnosti `ADATU` a `BDATU`.
- CDS `C_FixedAssetWorklist` využívá parametr **P_KeyDate**.
- Tím se automaticky filtruje:
  - platnost záznamů,
  - historická data,
  - změny KFZKZ v čase.

**DCL časovou platnost neřeší**, řeší ji datový model.

---

## 9. „Záhadné záznamy navíc“ (DU5)

- Na DU5 se objevuje 8 záznamů navíc.
- Data na DU5 jsou neúplná / nečistá.
- Analýza se provede až na DU4 (kopie produkce).
- Vedeno jako **TODO**.

---

## 10. End‑to‑end tok

1. Fiori předá P_KeyDate.
2. CDS vyfiltruje majetek podle ANLZ k danému dni.
3. Standardní DCL aplikuje FI-AA oprávnění.
4. Z‑DCL aplikuje filtr podle KFZKZ.
5. Pokud KFZKZ není v roli → záznam se skryje.
6. Pokud je KFZKZ prázdné → zobrazí se jen při oprávnění na prázdnou hodnotu.

---

## 11. Shrnutí pro business

- Uživatel vidí **pouze majetek v objektech**, na které má roli.
- Prázdné KFZKZ se nezobrazují.
- Historie je podporována automaticky (ANLZ).
- Řešení je čisté, udržovatelné a bezpečné.
- Test je nutné provádět na ne‑developerských účtech.

---

## 12. Závěr

Řešení je plně funkční a sladěné s požadavky zákazníka.  
Dalším krokem je ověření na DU4 nad produkčními daty.
