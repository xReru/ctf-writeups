# rain\_05\_date

### 📝 Challenge Description

> rain posted on X that they visited a certain place on December 25, 2025 (JST). However, the image seems to have been taken on a different day in 2025. Please answer the actual date the photo was taken in YYYY/MM/DD format (JST).
>
> Flag Format: `SWIMMER{YYYY/MM/DD}`&#x20;

### 🔍 Investigation Process

#### 1. Identifying the "Red Flags"

The investigation began by examining rain's X account, known for travel and expedition posts. I located the post from December 25, 2025.

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

Upon closer inspection, two major discrepancies (red flags) appeared:

1. The Clock vs. The Timestamp: The analog clock in the background of the photo clearly shows 4:00 PM. However, the X post was published _earlier_ than 4:00 PM on December 25th. This paradox confirms the photo is a "throwback" and was not taken on the day of the post.
2. Environment: The lighting and crowd density suggested a specific event was occurring.

<figure><img src="../../.gitbook/assets/Screenshot 2026-01-18 190435.png" alt=""><figcaption></figcaption></figure>

#### 2. Geolocation & Landmark Analysis

The location was easily identifiable by the distinct architecture in the background: Osaka-jō Hall in Osaka, Japan.

#### 3. Visual Intelligence: The Poster

I zoomed into a blurry poster visible on a bulletin board in the background. Despite the low resolution, a green figure and the text "10000 Freude" were discernible.

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

I performed a targeted Google search for "10000 Freude Osaka-jo Hall 2025".

While inspecting the background elements, I noticed a bulletin board with a distinct green poster featuring a man. Despite the blur, the layout was recognizable.

<div align="center"><figure><img src="../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure></div>

We confirmed that it matches the event which is Suntory 10000 Freude

<figure><img src="../../.gitbook/assets/image_2026-01-18_191343749.png" alt=""><figcaption></figcaption></figure>

#### 4. Event Verification

The search results confirmed that "Suntory 10,000 Freude" (a massive performance of Beethoven's 9th Symphony) is a recurring event at Osaka-jō Hall.

<figure><img src="../../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

By checking the official 2025 schedule for the "10,000 Freude" event:

* Event Date: December 7, 2025.
* Context: The presence of the posters and the specific crowd timing aligned perfectly with this date.

***

### 💡 Analysis & Solution

While the post was made on Christmas Day, the visual evidence (the clock, the specific event posters, and the event schedule) proves the photo was taken during the "10,000 Freude" performance earlier that month.

#### 🚩 Final Flag

`SWIMMER{2025/12/07}`

***

#### 🛡️ OSINT Pro-Tip

Recognizing Branding: In OSINT, colors and silhouettes are often enough to identify an event. The green poster is iconic for this specific Osaka event. When you see a unique color palette on a poster, search for "\[Location] + \[Color] + \[Event/Poster]" to narrow down the date.

**Credit**: A huge shoutout to my partner ([@dwyushi](https://www.instagram.com/dwyushi/)), who was the real MVP for this challenge! She has an incredible eye for detail and spotted the critical clues—including the poster details—that I completely missed. This solve is 100% thanks to her.

If you have any questions feel free to dm me [xreru](https://app.gitbook.com/u/sV63NjWn0kbva4C066LUjfLD3y92 "mention") or in [linkedin](https://www.linkedin.com/in/reru/)
