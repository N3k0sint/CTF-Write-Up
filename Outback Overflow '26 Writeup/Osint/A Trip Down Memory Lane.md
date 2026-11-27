# A Trip Down Memory Lane — Outback Overflow '26 Writeup
<img width="452" height="527" alt="image" src="https://github.com/user-attachments/assets/ee8392d7-e1c7-4764-8cc9-672144b6fca4" />

* **Competition:** Outback Overflow '26
* **Category:** OSINT (Open-Source Intelligence)
* **Points:** 75
* **Provided File:** `A_Trip_Down_Memory_Lane.txt`
* **Flag:** `OO{seafordline}`

---

## Challenge Prompt

> I remember when I was younger, I used to take the exact same train route through Adelaide every morning. I’ve always had an eye for art, so I remember the street art at the stop I’d get on and the stop I’d get off. I’ve described it below. Can you find it for me? Enter the flag as the number of the train line e.g. OO{oxfordline}
>
> I would get on at a stop with two main murals. The first at this station had a white cockatoo with a yellow crest, next to a red protea. The background was green. The second mural, done by the same artist, was of someone holding a yellow skateboard with some stickers on it.
>
> The stop I would get off at had a mural made by Ink and Ruby Studios. It was colourful and made of lots of abstract shapes.

---

## 1. Clue Extraction & OSINT Strategy

Breaking down `A_Trip_Down_Memory_Lane.txt` provides four key pivot points:

| Clue Type | Detail from Prompt | OSINT Pivot Value |
| :--- | :--- | :--- |
| **City / Network** | *"train route through Adelaide"* | Narrows search to **Adelaide Metro** (South Australia) rail stations. |
| **Stop 1 (Boarding)** | Two murals by the **same artist**: (1) **white cockatoo with yellow crest + red protea + green background**; (2) **someone holding a yellow skateboard with stickers** | Specific visual descriptors to match against Adelaide station public art catalogs. |
| **Stop 2 (Alighting)** | **Ink and Ruby Studios** mural (*"colourful and made of lots of abstract shapes"*) | **Highest-value pivot** contains an exact studio name. |
| **Flag Format** | `e.g. OO{oxfordline}` | Prefix `OO{...}`, all lowercase, no spaces, ending in `line`. |

---

## 2. Step-by-Step Solution

### Step 1: Identify the Alighting Station via "Ink and Ruby Studios"
Searching for `"Ink and Ruby Studios" Adelaide train station mural` immediately leads to two official sources:

1. **Adelaide Metro - Rail Care Public Art Directory** (`https://www.adelaidemetro.com.au/about-us/rail-care/public-art`)
   * Under its commissioned station murals list, the directory explicitly records: **`Ink & Ruby Studios at Brighton Railway Station`**.
2. **Tutti Arts - Brighton Railway Station Mural Feature (6 May 2026)** (`https://tutti.org.au/about/news/brighton-railway-station-mural/`)
   * Confirms that designer **Ella Simpson** from **Ink and Ruby Studios** collaborated with four **Tutti Arts** artists (**Nathan Dolphin**, **Nathan Smith**, **Banjobe Hunt**, and **Jacko Hooper**) and the Department for Infrastructure and Transport's **Rail Care** team.
   * Together, they transformed the **signal hut at Brighton Railway Station** into a vibrant, multi-layered abstract collage of cut-up printed artworks and painted shapes.

**Station Identified (Alighting):** **Brighton Railway Station**

---

### Step 2: Identify the Boarding Station via the Two Murals
Reviewing the **Adelaide Metro Public Art directory** (`https://www.adelaidemetro.com.au/about-us/rail-care/public-art`) for other stations along the same transit network reveals:
* `Mike Sanders at Noarlunga Railway Sation` *(listed twice on the page for his two works at the station)*.

Cross-referencing artist **Mike Sander** (`@mike_sander_art`) at **Noarlunga Railway Station / Noarlunga Centre Interchange** confirms both murals described in the prompt:

1. **Mural 1 - White Cockatoo & Red Protea on a Green Background:**
   * Documented in the **Facebook Reel by Morris Green** (`https://www.facebook.com/reel/1309638270764597`), titled **`N O A R L U N G A L I N E`** (*"a collaborative effort with @mike_sander_art at Noarlunga"*).
   * The video shows the large-scale realistic native flora and fauna mural at Noarlunga station, depicting a white sulphur-crested cockatoo beside a red protea flower on a green backdrop.
2. **Mural 2 - Person Holding a Yellow Skateboard with Stickers:**
   
   <img width="402" height="475" alt="image" src="https://github.com/user-attachments/assets/ef9cfe27-0d87-48a7-a2c2-704405868bae" />

   * Featured directly in the gallery on the **Adelaide Metro Public Art page** (`https://www.adelaidemetro.com.au/about-us/rail-care/public-art`).
   * Shows a close-up realistic mural of a skater gripping a bright **yellow skateboard deck** decorated with assorted skate stickers.

**Station Identified (Boarding):** **Noarlunga Centre Interchange (Noarlunga Railway Station)**

---

### Step 3: Map the Route Between Both Stations
Consulting the **Seaford line route documentation** (`https://en.wikipedia.org/wiki/Seaford_line`) confirms how a morning commuter travels between these two stations:

* **Boarding:** **Noarlunga Centre** (in Adelaide's southern suburbs, one stop north of Seaford Meadows and the Seaford terminus).
* **Direction of Travel:** Northbound toward **Adelaide Railway Station** (the CBD).
* **Alighting:** **Brighton** (located along the coast further north on the exact same rail corridor).
* **Train Line Name:** **Seaford line** (Adelaide Metro route code: **`SEAFRD`**).

---

## 3. Why the Flag Is `OO{seafordline}`

Three details explain why **`OO{seafordline}`** is the exact flag—and why alternative guesses fail:

1. **Why a name instead of a number?**
   * The prompt states *"Enter the flag as the number of the train line e.g. OO{oxfordline}"*. While Adelaide Metro assigns route numbers to buses, its **suburban train lines are identified by the name of their outer terminus** (e.g., *Seaford line*, *Gawler line*, *Belair line*, *Outer Harbor line*, *Grange line*, *Flinders line*).
   * The example **`OO{oxfordline}`** clarifies the exact string structure required: lowercase, no spaces, with `line` appended.
2. **Why `seafordline` instead of `noarlungaline`?**
   * The Facebook reel is captioned `N O A R L U N G A L I N E` because the corridor was historically called the **Noarlunga Centre line** up until February 2014, when the 5.7 km electrified southern extension opened to Seaford and officially renamed the entire route the **Seaford line**.
   * Because the Brighton signal hut mural was completed in **2026**, the corridor's modern official name—**Seaford line**—is the canonical answer.
3. **Final Formatting:**
   * `Seaford line` -> lowercase & strip spaces -> `seafordline` -> wrap in flag format -> **`OO{seafordline}`**.

---

## 4. Visual Evidence Summary

* **Skateboard Mural (Noarlunga Centre Interchange):** Close-up mural by **Mike Sander** showing a skater holding a yellow skateboard covered in stickers (documented on the Adelaide Metro Rail Care Public Art page).
* **Cockatoo & Protea Mural (Noarlunga Centre Interchange):** Large native wildlife and flora mural by **Mike Sander** featuring a sulphur-crested white cockatoo beside a red protea on a green background (documented in Morris Green's Facebook Reel).
* **Signal Hut Mural (Brighton Railway Station):** Colourful abstract geometric collage mural by **Ella Simpson (Ink and Ruby Studios)** and **Tutti Arts** (documented on Tutti Arts News and Adelaide Metro Public Art).
* **Train & Route Map (Adelaide Metro Seaford Line):** Electrified `SEAFRD` rail corridor operated by Adelaide Metro **4000 Class EMU** trains connecting **Noarlunga Centre** and **Brighton**.

---

## 5. Reference Links

* **Adelaide Metro — Rail Care Public Art:** https://www.adelaidemetro.com.au/about-us/rail-care/public-art
* **Tutti Arts — Brighton Railway Station Mural:** https://tutti.org.au/about/news/brighton-railway-station-mural/
* **Facebook Reel — Noarlunga Station Cockatoo & Protea Mural:** https://www.facebook.com/reel/1309638270764597
* **Wikipedia — Seaford Line (Adelaide):** https://en.wikipedia.org/wiki/Seaford_line
