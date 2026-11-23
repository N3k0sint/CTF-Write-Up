# CTF Writeup: Where Did My Mate Go?
<img width="450" height="426" alt="image" src="https://github.com/user-attachments/assets/487a9cdd-bac0-42ef-98f2-44fb3907f9f0" />

| Field | Value |
|---|---|
| **Category** | OSINT |
| **Difficulty** | Medium |
| **Points** | 200 |
| **Location** | Manly Beach Promenade, opposite Hotel Steyne and The Corso, New South Wales, Australia |
| **Flag Format** | `oo{lat,long}` (4 decimal places) |
| **Final Flag** | `oo{-33.7958,151.2879}` |

---

## Challenge Overview
<img width="708" height="704" alt="p2" src="https://github.com/user-attachments/assets/44bde47e-ed1b-4cce-8303-dc967ed33d8a" />

The challenge requires identifying the real-world location shown in a reference image or Street View frame. The visual clues point to Manly Beach in Sydney, Australia.

## Step-by-Step Visual Analysis

### 1. Initial Regional Identification (Google Reverse Images)

We are using Google Image search to reverse images and results shown

<img width="1420" height="687" alt="image" src="https://github.com/user-attachments/assets/7885cb3d-e7dd-4bc4-9beb-e297ccbe2d90" />

The wide sandy shoreline, rolling Pacific surf, patterned promenade pavers, and low sandstone-capped seawall immediately point to Manly Beach in Sydney, Australia.

Looking out toward the ocean (east-northeast), the Queenscliff and Freshwater headland is visible on the far-left horizon. This confirms that the camera faces roughly ENE, approximately 68 degrees along the Manly beachfront.


### 2. Narrowing Down the Exact Stretch of Promenade

Click on the third link (Trip.com): 
```text
https://www.trip.com/moments/detail/manly-2126991-146213314/`
```


<img width="1320" height="870" alt="image" src="https://github.com/user-attachments/assets/f2da2b6d-e804-4634-8639-aa8a3b9c519c" />


- **Foreground seawall and paving:** A herringbone brick footpath bordered by a weathered, beveled sandstone coping wall runs along the Manly oceanfront promenade.
- **Midground beach markers:** A yellow beach-safety sign stands to the left of center.
- **Stormwater features:** A pair of parallel stormwater outfall pipes with two tall vertical marker poles appears in the swash zone to the right of center.
- **Shoreline details:** A surfer walks along the sand on the left, while the bow of a white surf craft rests on the sand near the far-right edge.

### 3. Pinpointing the Exact Google Street View Panorama

Dropping into Google Street View along the Manly Beach footpath directly opposite Hotel Steyne and The Corso, using the October 2024 imagery, matches every visual detail. These details include the surfer, yellow warning sign, twin pipes, white boat edge, and exact cloud formation in the sky.

Aligning the camera heading and pitch reproduces the challenge frame:

- **Heading:** 67.72 degrees
- **Pitch:** 82.9 degrees

## Exact Clues and Reference Link

The following Street View link is limited to the challenge coordinates:

```text
https://www.google.com/maps/@-33.7958525,151.2879171,3a,75y,67.72h,82.9t/data=!3m6!1e1!3m4!1sZFreLBUYgP7APRiQr_DvUg!2e0!7i16384!8i8192
```
<img width="864" height="864" alt="image" src="https://github.com/user-attachments/assets/dab2e6ad-e680-4f72-b0ee-f91076f454f3" />

Or open the location directly:

[Open the Google Street View Link](https://www.google.com/maps/@-33.7958525,151.2879171,3a,75y,67.72h,82.9t/data=!3m6!1e1!3m4!1sZFreLBUYgP7APRiQr_DvUg!2e0!7i16384!8i8192)


### Extracted Panorama Details

| Detail | Value |
|---|---|
| Panorama ID | `ZFreLBUYgP7APRiQr_DvUg` |
| Imagery Date | October 2024 |
| Camera Latitude | `-33.7958525` |
| Camera Longitude | `151.2879171` |
| Camera Heading (Yaw) | `67.72 deg` |
| Camera Tilt (Pitch) | `82.9 deg` |

## Coordinate Formatting and Flag Derivation

The raw Street View coordinates are:

- **Latitude:** `-33.7958525`
- **Longitude:** `151.2879171`

To match the required four-decimal-place format, the values are truncated rather than rounded:

- **Latitude:** `-33.7958`
- **Longitude:** `151.2879`

The challenge uses the `oo{lat,long}` format:

```text
oo{-33.7958,151.2879}
```

**Final Flag:** `oo{-33.7958,151.2879}`
