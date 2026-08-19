---
name: clickfunnels-page-markup
description: >
  Reference for PML (Page Markup Language), the XML-like DSL that ClickFunnels 2.0
  uses to describe page content. Read this skill when authoring or editing markup
  for any surface that accepts a `markup` field - currently pages and blog posts.
  Writing `markup` replaces the entire page and PML covers only a subset of what
  the ClickFunnels editor can build, so read the approval guardrail before
  touching a page that already exists.
  This is a language reference; for the HTTP APIs that consume it, see the
  Pages Skill (for funnel/site pages) or the Blogs Skill (for blog posts).
version: "1.0"
author: ClickFunnels
tools: []
references:
  - path: /llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: /.well-known/pages/skill.md
    description: The Pages API - read this for authentication, the page CRUD endpoints, and the checkout-products attachment shape.
  - path: /.well-known/blogs/skill.md
    description: The Blogs API - read this to create or update blog posts whose body is PML.
  - path: /.well-known/funnels/skill.md
    description: Funnel workflow APIs (split tests, conditional splits) that compose pages built with this DSL.
---

# ClickFunnels Page Markup Language (PML)

PML is an XML-like DSL compiled by the CF2 page engine into the full page tree the
editor renders. It is the canonical authoring format for both funnel/site pages
(via the Pages API) and blog posts (via the Blogs API).

This document is a **language reference** - elements, attributes, design system,
patterns, common errors. It contains no HTTP details. For that, follow the
references above.

## When to read this skill

- Authoring markup for a new page or blog post from scratch
- Editing existing markup returned by `expand[]=markup` on any surface
- Debugging a `400 Markup is not valid PML` response
- Designing a new conversion-oriented page and choosing fonts, colors, gradients
- Composing a multi-section page that follows CF2's design system

## STOP: get approval before overwriting an existing page

PML covers a subset of what the ClickFunnels page editor can build. Four
properties of that subset make writing `markup` to an existing page dangerous:

- **Reading is lossy.** `expand[]=markup` runs the stored page through a
  serializer that has an equivalent for the elements documented in this skill
  and nothing else. Any other element the editor supports is dropped from the
  string you get back: the element, its settings and its styling are simply
  absent. There is no error, no warning, and no placeholder marking the hole.
- **An empty read does not mean an empty page.** Because dropped elements leave
  no trace, a page built entirely from elements PML cannot express reads back as
  an empty string. `""` is therefore not evidence that the page is blank, and it
  is not a reason to skip the approval below.
- **A failed read looks like content.** If the read comes back as
  `<!-- markup unavailable: page tree could not be serialized -->`, the
  serializer could not handle the page at all. Treat the page as unreadable and
  do not write. Sending that string back replaces the page with nothing.
- **Writing replaces everything.** Sending `markup` rebuilds the page from your
  string and overwrites both the published page and the draft the editor opens.
  Nothing is merged, and there is no partial write.

Put together, a read-modify-write cycle on a page a person built in the editor
silently destroys whatever PML could not express, in the live page and in the
editor, and the API cannot put it back.

**The rule:** if the page or post could have been created or edited in the
ClickFunnels editor, STOP and ask the person for explicit approval before you
send `markup`. Tell them plainly that you are replacing the whole page and that
editor-only content will not survive. A general instruction such as "update the
page" or "improve the headline" is NOT that approval.

You do not need to ask when:

- you are creating a brand new page or post and passing `markup` on create
- you created the page through the API in this session and nobody has opened it
  in the editor since

If you cannot tell how a page was authored, assume the editor built it and ask.
When you have an additive route, prefer it: create a new page and position it,
rather than rewriting a page that is already live and converting. If approval is
refused, the edit belongs in the editor, not in this API.

## DSL Quick Rules

```
<section>          <- full-width band
  <row cols="N">   <- horizontal split (1-4 columns for content)
    <column>       <- content container (NO padding attribute)
      <headline fontsize="72px" weight="black" color="#fff" font="Poppins">
        Text with <span color="#60a5fa">accent</span>
      </headline>
      <paragraph size="m" color="#94a3b8" leading="relaxed" font="Inter">
        Body copy here.
      </paragraph>
      <button bg="linear-gradient(135deg, #2563eb 0%, #7c3aed 100%)"
              color="#fff" radius="full" align="center">CTA -></button>
    </column>
  </row>
</section>
```

**Critical nesting:** section -> row -> column -> content. Never skip a level.
**Colors:** #hex only. No rgba(), no named colors.
**fontsize:** Must include unit - "72px" not "72".

## Choosing a Design Style

Six canonical styles (full hex palettes in the DSL reference below):

| Style             | Use when                                  | Fonts                        |
|-------------------|-------------------------------------------|------------------------------|
| modern_tech       | Software, apps, developer tools           | Poppins + Inter              |
| luxury_premium    | High-end services, coaching, consulting   | Playfair Display + Lato      |
| bold_striking     | High-urgency, sales, attention-grabbing   | Anton + Roboto               |
| minimalist_clean  | Clean SaaS, B2B, professional services   | Work Sans + Work Sans        |
| playful_creative  | Consumer apps, youth, creative services  | Montserrat + Open Sans       |
| warm_inviting     | Health, wellness, coaching, local biz    | Libre Baskerville + Roboto   |

## Standard Page Structure

For a sales/landing page (most common):

1. **Hero** - big headline, subheadline, single CTA (padding 100-120px)
2. **Stats strip** - 4-col numbers + labels (padding 0 0)
3. **How it works / Features** - 3-col cards
4. **Benefits split** - 2-col with lists
5. **Final CTA** - gradient section, mirror hero CTA

## Example (minimal valid markup)

```xml
<section bg="#0f172a" padding="100 40">
  <row>
    <column>
      <headline fontsize="72px" weight="black" align="center"
                color="#ffffff" font="Poppins">
        Your Headline <span color="#3b82f6">Here</span>
      </headline>
      <paragraph size="l" color="#94a3b8" align="center" leading="relaxed">
        Supporting copy in one or two sentences.
      </paragraph>
      <button bg="linear-gradient(135deg, #2563eb 0%, #7c3aed 100%)"
              color="#fff" fontsize="18px" radius="full" align="center">
        Call to Action ->
      </button>
    </column>
  </row>
</section>
```

## Common Validation Errors

These are the DSL parser errors returned at validate time. HTTP-level errors
(401, 404, etc.) live in the consuming API skill (Pages or Blogs).

| Error | Cause | Fix |
|-------|-------|-----|
| 400 "Markup is not valid PML: unknown top-level element(s)" | Unknown DSL tag at the top level | Check element name in the DSL reference below |
| 400 "Markup is not valid PML: no PML elements found" | Plain text or raw HTML in the `markup` field | Wrap content in valid PML elements (`<section>`, `<saved-section>`, etc.) |
| 400 "`<column>` element must be inside a `<row>`" | Nesting violation | Always nest section -> row -> column -> content |
| 400 "Invalid attribute(s) '...' on `<element>`" | Attribute is not allowed on that element (e.g., `margintop` on `<input>`) | Check the per-element attribute lists below |
| 400 "Markup appears to be truncated: ..." | The `markup` you sent stops mid-construct - a tag, attribute value, comment or `<customhtml>` body never closes, or elements are still open at the end of the document. Almost always means your markup was cut off before you finished writing it (an output token limit, a truncated buffer). | Re-send the complete document. Do NOT retry with the same truncated string, and do NOT try to "repair" it by sending only the tail - `markup` replaces the whole page, so a partial document deletes everything after the cut-off point. If the page is long, add the pixel or code with `head_code`/`footer_code` instead of rewriting the body. |

================================================================================
## 2. DSL OVERVIEW
================================================================================

  PML is an XML-like format compiled by the CF2 page engine into a full
  page tree. After a successful `POST`/`PATCH` to the consuming API, the
  page is immediately live.

### Mandatory Nesting Rules

  Every <section> must contain >=1 <row>
  Every <row>    must contain >=1 <column>
  All content elements must live inside a <column>

  WRONG (400 error):
    <section><headline>Text</headline></section>
    <section><column>...</column></section>

  CORRECT:
    <section>
      <row>
        <column>
          <headline>Text</headline>
        </column>
      </row>
    </section>

### Page Structure

  A page is a flat sequence of <section> elements.
  Each section is a full-width horizontal band.
  Sections stack vertically top to bottom.


================================================================================
## 3. STRUCTURE ELEMENTS
================================================================================

### <section>

  Full-width container. Top-level page band.

  Attributes:
    bg              #hex color or CSS gradient string
    bgimage         Background image URL
    bgprompt        AI-generated background - describe in plain English
                    e.g. bgprompt="dark geometric grid pattern, subtle blue glow"
    padding         "vertical horizontal" px. Default "80 20". Single value = all sides.
    padding-mobile  Same format, applies at <=700px. Default "56 16"
    shadow          sm | md | lg
    radius          sm(4px) | md(8px) | lg(16px) | xl(24px)
    border          thin | medium | thick
    opacity         0-100
    id              CSS/JS targeting

  Examples:
    <section bg="#0f172a" padding="100 40">
    <section bg="linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%)" padding="120 40">
    <section bg="midnight" padding="80 20">  <- gradient preset by name

### <row>

  Horizontal column divider within a section.

  Attributes:
    cols            1-12. Omit or use 1 for single centered column.
    padding         Same format as section
    padding-mobile
    id

  Typical values:
    <row>           single centered column
    <row cols="2">  two-column (50/50)
    <row cols="3">  three-column feature grid
    <row cols="4">  four-column stat strip

### <column>

  Content container. NO padding attribute (use section/row padding instead).

  Attributes:
    bg              Background color or gradient (for card-style columns)
    shadow          sm | md | lg
    radius          sm | md | lg | xl
    border          thin | medium | thick
    opacity         0-100
    marginhorizontal        px (left/right buffer on column content)
    marginhorizontal-mobile
    id

### <flex>

  Flexbox for horizontal layouts only. Place inside <column>.

  Attributes:
    direction       Always "row" (do NOT use "column" - that's a layout antipattern here)
    justify         flex-start | center | flex-end | space-around | space-between | space-evenly
    align           flex-start | center | flex-end | baseline | stretch
    gap             px between items. e.g. "16"
    width           "100%" | "400px"
    height          "auto" | "60px"
    wrap            nowrap | wrap | wrap-reverse
    padding         Same format as section
    margintop       px spacing above the flex container (also: margintop-mobile)

  Mobile variants: direction-mobile, justify-mobile, align-mobile, wrap-mobile

  Example:
    <flex direction="row" justify="center" gap="16" wrap="nowrap">
      <button bg="#6366f1">Primary</button>
      <button bg="transparent" color="#6366f1">Secondary</button>
    </flex>


================================================================================
## 4. TEXT ELEMENTS
================================================================================

  All text elements share:
    size        xl(48px) | l(40px) | m(32px) | s(24px)
    fontsize    Custom size, overrides preset. Must include unit: "72px" "5rem" "100px"
    align       left | center | right
    color       #hex only (no rgba, no named colors)
    font        Google Font name (see full list below)
    weight      thin|light|normal|medium|semibold|bold|extrabold|black
    transform   uppercase | lowercase | capitalize
    leading     tight | snug | normal | relaxed | loose
    spacing     tight | normal | wide | wider | widest
    margintop   px spacing above this element
    margintop-mobile
    id

### Supported Google Fonts

  DISPLAY / HEADLINE (character + impact):
    Playfair Display   - editorial elegance, serif
    Cormorant Garamond - refined luxury, thin serif
    DM Serif Display   - warm sophistication, serif
    Fraunces           - expressive, variable-weight serif
    Space Grotesk      - modern tech, geometric sans
    Outfit             - clean geometric, versatile
    Syne               - bold contemporary, display sans
    Bebas Neue         - high impact, condensed caps
    Oswald             - strong, condensed sans
    Anton              - maximum impact, display caps
    Lobster            - playful script energy
    Pacifico           - friendly, rounded script
    Libre Baskerville  - warm, readable serif
    Poppins            - versatile, rounded geometric
    Montserrat         - professional, geometric
    Merriweather       - warm, readable serif

  BODY / READABLE:
    Inter              - neutral, highly readable
    Work Sans          - clean, modern
    Source Sans Pro    - clean professionalism
    Lato               - approachable, humanist
    Nunito             - friendly, rounded
    Quicksand          - geometric, friendly
    IBM Plex Sans      - technical, modern
    Open Sans          - universal readability
    Roboto             - neutral, versatile

  FONT PAIRINGS (proven combinations):
    Playfair Display + Lato          -> classic editorial elegance
    Poppins + Inter                  -> modern product/tech
    Space Grotesk + IBM Plex Sans    -> developer/tech aesthetic
    DM Serif Display + Source Sans Pro -> warm consultative
    Bebas Neue + Roboto              -> bold, high-energy
    Montserrat + Open Sans           -> professional versatile
    NEVER pair fonts with similar character shapes (e.g., Roboto + Open Sans)

### <headline>

  Primary heading. Size xl-s or override with fontsize.

  <headline fontsize="80px" weight="black" align="center" color="#fff" font="Poppins">
    Build Pages <span color="#a78bfa">That Convert</span>
  </headline>

### <subheadline>

  Secondary heading. Sits below headline.

  <subheadline size="l" color="#94a3b8" align="center" weight="medium" font="Inter">
    The API that writes itself
  </subheadline>

### <paragraph>

  Body copy. Use leading="relaxed" for comfortable reading.

  <paragraph size="m" color="#64748b" align="center" leading="relaxed">
    One HTTP call. DSL markup in. Full rendered page out.
  </paragraph>

### Inline Formatting

  Works inside all text elements:
    <b>bold</b>
    <i>italic</i>
    <span color="#f59e0b">colored text</span>
    <span bg="#fef3c7" color="#92400e">highlighted</span>
    Line One<br/>Line Two

### Text encoding (non-ASCII characters)

  Type accented letters and punctuation as their literal UTF-8 characters
  (umlauts, accents, em dash, curly quotes, arrows). Do NOT use HTML character
  entities such as &mdash;, &rarr;, &nbsp;, or &amp; inside text - they are not
  decoded on a page and render literally (e.g. "&mdash;" shows up verbatim) or
  are dropped entirely. Write the em dash character itself, "und" instead of
  "&amp;", and so on. (Raw HTML inside <customhtml> is the one exception - there,
  normal HTML entities work.)


================================================================================
## 5. MEDIA ELEMENTS
================================================================================

### <image>

  Attributes:
    src       Hosted image URL (for existing images)
    prompt    AI-generated image - write a descriptive scene/mood prompt
    alt       Alt text for accessibility
    align     left | center | right
    width     "50%" | "300px" | "100%"  (prefer % for responsiveness)
    height    "auto" | "300px"
    radius    sm | md | lg | xl | full | "20px"

  Prompts - be specific for quality:
    WEAK:   prompt="person smiling"
    STRONG: prompt="professional woman in her early 40s, warm confident smile,
                    soft natural light, modern office background, editorial style"

  Include: lighting, emotion, environment, style direction.

  Examples:
    <image src="https://cdn.example.com/hero.jpg" width="100%" radius="lg"/>
    <image prompt="abstract dark navy geometric wave, subtle blue glow" width="100%"/>
    <image prompt="confident male entrepreneur at laptop, golden hour light, rooftop" width="60%" radius="md"/>

### <video>

  Auto-detects YouTube, Vimeo, Wistia, HTML5 video.

  <video src="https://www.youtube.com/watch?v=XXXXXXXXXXXX"/>
  <video src="https://vimeo.com/XXXXXXXXX"/>

### <divider>

  Horizontal rule / spacing element.
  Attributes: color, padding (vertical space px), width ("50%"/"100%"), align

  <divider color="#1e293b" padding="40" width="100%"/>


================================================================================
## 6. INTERACTIVE ELEMENTS
================================================================================

### <button>

  Attributes:
    bg            Solid #hex, gradient string, or gradient preset name
                  Presets: sunset | ocean | forest | fire | midnight | royal | peach | sky
    color         Text color (#hex)
    font          Google Font name
    fontsize      "18px" etc.
    fontsize-mobile
    radius        sm | md | lg | xl | full | "8px"
    align         left | center | right
    style-guide   style1 | style2 | style3 (visual preset themes)
    href          URL to link the button to; renders as an anchor. Use for a
                  navigation CTA (e.g. "Book a call", a link to another page, or
                  an in-page anchor like "#pricing"). Do NOT set href on an
                  opt-in / form-submit button - that button submits the form.

  CTA best practices:
    - Use high-contrast: if bg is dark, accent should POP (orange, coral, bright blue)
    - radius="full" performs well on most pages
    - Use action verbs: "Get Started ->" "Start Free Trial" "Download Now"
    - One CTA per section. Two CTAs = no CTA.

  Examples:
    <button bg="linear-gradient(135deg, #7c3aed 0%, #2563eb 100%)"
            color="#fff" font="Poppins" fontsize="18px" radius="full" align="center">
      Get Started Free ->
    </button>
    <button bg="sunset" color="#fff" radius="lg" align="center">Learn More</button>

### <input>

  Attributes:
    label         Field label
    type          name | first_name | last_name | email | phone_number |
                  shipping_address | shipping_city | shipping_state |
                  shipping_country | shipping_zip | text
    placeholder   Hint text
    required      "true" | "false"
    width         "100%" | "300px"
    radius        sm | md | lg | "8px"
    border        thin | medium | thick | "2 solid #ccc"
    padding       px

  ! **`<input>` does NOT accept `margintop` / `margintop-mobile`** - those
     attributes are content-element-only (headline, paragraph, button,
     image, list, accordion, countdown). Sending one on `<input>` returns
     400 "Invalid attribute(s) 'margintop' on <input>". Wrap inputs in a
     `<flex gap="...">` container to space them apart, or stack them in
     separate `<column>`s.

  <input label="Email Address" type="email" placeholder="you@example.com"
         required="true" width="100%" radius="md"/>

### <link>

  <link href="https://example.com">Link text here</link>


================================================================================
## 7. COMPLEX ELEMENTS
================================================================================

### <list>

  Icon list for benefits, features, bullet points.

  Attributes:
    icon        check | star | heart | arrow | plus | bolt | thumbsup | circle
    iconcolor   #hex for icon color
    size        s | m | l
    color       Text color
    font        Google Font name

  <list icon="check" iconcolor="#10b981" size="m" color="#e2e8f0" font="Inter">
    <item>First benefit - be specific, not vague</item>
    <item>Second benefit - outcomes beat features</item>
    <item>Third benefit - specifics beat generalities</item>
  </list>

### <countdown>

  Live countdown timer. Supports fixed date or evergreen (resets per visitor).

  Attributes:
    type          countdown | evergreen
    date          "YYYY-MM-DD" (countdown type)
    time          "HH:MM:SS" (countdown type)
    timezone      e.g. "America/New_York"
    bg            Background color
    padding       px (vertical)
    paddinglr     px (left/right)
    radius        sm | md | lg
    numbercolor   #hex
    numberfont    Google Font name
    numbersize    px
    numberweight  bold etc.
    labelcolor    #hex
    labelfont     Google Font name
    labelsize     px
    labelweight

  <countdown type="countdown" date="2026-03-31" time="23:59:59"
             bg="#0f172a" radius="md" padding="20"
             numbercolor="#f59e0b" numberfont="Poppins" numbersize="48" numberweight="black"
             labelcolor="#64748b" labelsize="14"/>

### <accordion>

  Collapsible FAQ sections.
  Attributes: bg, textcolor, margintop

  <accordion bg="#0f172a" textcolor="#e2e8f0">
    <item title="Question goes here?">Detailed answer goes here.</item>
    <item title="Another question?">Another answer here.</item>
  </accordion>

### <customhtml>

  Raw HTML/CSS/JS injection. Use ONLY for things DSL cannot do.

  DO NOT use for: text, buttons, images, lists, cards, layout, spacing.
  Only use for: custom widgets, third-party embeds, CSS animations.

  <customhtml>
    <style>
      .badge { background: #7c3aed; color: #fff; padding: 4px 12px;
               border-radius: 99px; font-size: 12px; display: inline-block; }
    </style>
    <span class="badge">SALE</span>
  </customhtml>

### <saved-section>

  Reference a saved/universal section by ID.
  <saved-section ref="SECTION_ID"/>


================================================================================
## 8. COMMERCE ELEMENTS
================================================================================

  Use on checkout, order, and product pages only. For attaching products to a
  `<checkout/>` element, see the Pages Skill - that surface is owned by the
  consuming API, not the DSL.

  <checkout/>                              Full payment/checkout form
                                           Supports steps="1|2|3" (default 1)
                                           Renders the products attached to the
                                           page's show_page_step.
  <order-summary linked-checkout-id="N"/> Cart + totals (link to checkout id)
  <confirmation/>                          Order confirmation display
  <product-carousel/>                      Product image carousel
  <product-name/>                          Dynamic: {{product.name}}
  <product-description/>                   Dynamic: {{product.description}}
  <product-price/>                         Dynamic: {{product.price}}
  <product-image/>                         Dynamic: product hero image
  <product-link/>                          Dynamic: product URL


================================================================================
## 9. GRADIENT PRESETS
================================================================================

  Use as bg value on <section>, <column>, or <button>:

  Name        Approx direction   Colors
  ---------------------------------------------------------
  sunset      135deg             #ff512f -> #dd2476  (red-pink)
  ocean       135deg             #2193b0 -> #6dd5ed  (blue-cyan)
  forest      135deg             #134e5e -> #71b280  (teal-green)
  fire        135deg             #f7971e -> #ffd200  (orange-yellow)
  midnight    135deg             #232526 -> #414345  (dark grey)
  royal       135deg             #141e30 -> #243b55  (deep navy)
  peach       135deg             #ed8966 -> #ffb347  (peach-orange)
  sky         180deg             #87ceeb -> #ffffff  (sky-white)

  Custom gradient format:
    "linear-gradient(135deg, #6366f1 0%, #8b5cf6 50%, #ec4899 100%)"
    "linear-gradient(160deg, #030712 0%, #0F0826 50%, #030712 100%)"


================================================================================
## 10. DESIGN SYSTEM - TYPOGRAPHY
================================================================================

  Typography is the primary design lever. Generic fonts kill conversions.
  Commit to a font pairing and use it consistently throughout the page.

### Typography Hierarchy

  HERO HEADLINE    fontsize="80px-120px" weight="black" or "extrabold"
                   Font: Playfair Display / Poppins / Space Grotesk / Bebas Neue
  SECTION TITLE    size="l" or fontsize="40px-56px" weight="bold"
                   Font: same as hero headline
  CARD TITLE       size="s" or fontsize="22px-28px" weight="bold" or "semibold"
  EYEBROW LABEL    size="s" weight="bold" color="<accent>" align="center"
                   (small ALL-CAPS or styled label above the main headline)
  BODY COPY        size="m" or size="s" weight="normal" leading="relaxed"
                   Font: Inter / Source Sans Pro / Lato
  CAPTION          size="s" weight="medium" color="<muted>"

### Hierarchy Rule

  NEVER assign the same visual weight to elements at different hierarchy levels.
  Hero headline -> section title -> card title -> body must each step DOWN noticeably.

### Font Selection by Aesthetic

  Luxury Editorial:    Playfair Display / Cormorant Garamond + Lato
  Modern Tech:         Space Grotesk / Poppins + Inter / IBM Plex Sans
  Bold Striking:       Bebas Neue / Anton + Roboto
  Warm Inviting:       Libre Baskerville / DM Serif Display + Source Sans Pro / Nunito
  Playful Creative:    Montserrat / Pacifico + Open Sans / Nunito
  Minimalist Clean:    Work Sans / Outfit + Work Sans (same family, different weights)


================================================================================
## 11. DESIGN SYSTEM - COLOR
================================================================================

### Color Architecture

  Every page needs three color roles:

  DOMINANT    bg color owning 60-70% of visual weight
  ACCENT      CTA color - must pop against everything (never blend in)
  NEUTRAL     Text, borders, muted elements

### Dark Page Palette Template

  hero_bg:      #0f172a (or #030712, #080C14, #1a1a2e)
  section_bg:   #0f172a or #111827
  surface/card: #1e293b
  border:       #334155
  text_primary: #ffffff or #f8fafc
  text_body:    #94a3b8
  text_muted:   #64748b
  accent:       ONE vivid color (see below)

### Light Page Palette Template

  hero_bg:      #ffffff or #f8fafc
  section_alt:  #f1f5f9
  surface/card: #ffffff
  border:       #e2e8f0
  text_primary: #0f172a or #111827
  text_body:    #374151 or #4b5563
  text_muted:   #9ca3af
  accent:       ONE vivid color

### Accent Color Psychology

  #3b82f6 (blue)       -> trust, professional, tech
  #6366f1 (indigo)     -> creative, premium, modern
  #7c3aed (violet)     -> innovative, bold, energetic
  #10b981 (emerald)    -> growth, health, positive
  #f59e0b (amber)      -> urgency, warmth, attention
  #ef4444 (red)        -> power, urgency, bold
  #f97316 (orange)     -> energy, friendly, enthusiasm
  #d4af37 (gold)       -> luxury, prestige, premium
  #ec4899 (pink)       -> playful, approachable, modern

  CTA RULE: Orange/coral CTAs outperform on most pages. Use high-contrast - the
  CTA must be instantly distinguishable from every other element on the page.

### Gradient Text (dark pages)

  Wrap span around headline text for gradient effect:
  <headline fontsize="80px" weight="black" color="#fff" font="Poppins">
    Build Funnels <span color="#a78bfa">That Convert</span>
  </headline>

  Note: span only supports solid color, not gradient. Use as accent within headline.


================================================================================
## 12. DESIGN SYSTEM - AESTHETIC STYLES
================================================================================

  These are the six canonical CF2 aesthetic styles. Use exact hex values for brand consistency.

  modern_tech
    hero_bg:   #0f172a    content_bg: #f8fafc    cta_bg: #1e293b
    accent:    #3b82f6    text_dark:  #ffffff     text_light: #1e293b
    muted:     #64748b
    fonts:     Poppins (headline bold) + Inter (body normal)
    effects:   shadow=md, radius=lg, gradient=midnight
    keywords:  modern tech professional clean digital software startup

  luxury_premium
    hero_bg:   #1a1a2e    content_bg: #faf5ef    cta_bg: #16213e
    accent:    #d4af37    text_dark:  #faf5ef     text_light: #1a1a2e
    muted:     #8b7355
    fonts:     Playfair Display (headline normal) + Lato (body light)
    effects:   shadow=sm, radius=md, gradient=royal
    keywords:  luxury premium elegant high-end exclusive sophisticated

  playful_creative
    hero_bg:   #7c3aed    content_bg: #ffffff    cta_bg: #f97316
    accent:    #f97316    text_dark:  #ffffff     text_light: #1e293b
    muted:     #6b7280
    fonts:     Montserrat (headline extrabold) + Open Sans (body normal)
    effects:   shadow=lg, radius=xl, gradient=sunset
    keywords:  playful fun creative bold energetic vibrant exciting

  minimalist_clean
    hero_bg:   #ffffff    content_bg: #f9fafb    cta_bg: #111827
    accent:    #2563eb    text_dark:  #ffffff     text_light: #111827
    muted:     #6b7280
    fonts:     Work Sans (headline semibold) + Work Sans (body normal)
    effects:   shadow=sm, radius=md, gradient=none
    keywords:  minimal minimalist clean simple sleek understated

  bold_striking
    hero_bg:   #000000    content_bg: #ffffff    cta_bg: #dc2626
    accent:    #ef4444    text_dark:  #ffffff     text_light: #000000
    muted:     #4b5563
    fonts:     Anton (headline normal) + Roboto (body normal)
    effects:   shadow=lg, radius=none, gradient=none
    keywords:  bold striking dramatic intense powerful edgy aggressive

  warm_inviting
    hero_bg:   #065f46    content_bg: #fffbeb    cta_bg: #92400e
    accent:    #10b981    text_dark:  #ffffff     text_light: #1c1917
    muted:     #78716c
    fonts:     Libre Baskerville (headline bold) + Roboto (body light)
    effects:   shadow=md, radius=xl, gradient=forest
    keywords:  warm inviting friendly welcoming natural organic earthy


================================================================================
## 13. DESIGN SYSTEM - SECTION PATTERNS
================================================================================

  Use these proven patterns to structure conversion pages.

### Hero Section

  The most critical section. If it fails, nothing else matters.

  Rules:
  - Minimum padding: 100px. Cramped heroes fail.
  - Headline: 72px+ minimum. If it doesn't feel massive, it's too small.
  - Single CTA. One action only.
  - Subheadline supports; never competes with headline

  Strategies - pick one:
    TYPOGRAPHY-LED:  Massive headline IS the visual (100px+, black weight)
    IMAGE-LED:       Hero image is the visual; text overlaid or adjacent
    SPLIT:           Two-column - text left, image right (natural eye flow)

  Template (typography-led):
    <section bg="<dark>" padding="120 40">
      <row>
        <column>
          <paragraph size="s" color="<accent>" weight="bold" align="center">
            * EYEBROW LABEL *
          </paragraph>
          <headline fontsize="88px" weight="black" align="center" color="#fff" font="Poppins">
            Main Headline<br/><span color="<accent>">Colored Part</span>
          </headline>
          <paragraph size="l" color="<muted>" align="center" leading="relaxed">
            Supporting copy. One or two sentences. Benefits, not features.
          </paragraph>
          <button bg="<accent-gradient>" color="#fff" radius="full" align="center" fontsize="18px">
            Primary Action ->
          </button>
        </column>
      </row>
    </section>

### Stats Strip

  4-column band of big numbers + labels. Often zero padding (full bleed).

    <section bg="<surface>" padding="0 0">
      <row cols="4">
        <column>
          <headline fontsize="52px" weight="black" align="center" color="<accent1>">10K+</headline>
          <paragraph size="s" weight="semibold" align="center" color="<muted>">Users</paragraph>
        </column>
        ... x 3 more
      </row>
    </section>

  Each stat column should use a DIFFERENT accent color for visual variety.

### Feature Cards (3-column)

    <row cols="3">
      <column bg="<surface>" radius="lg" shadow="md">
        <headline size="xl" align="center"></headline>
        <headline size="s" weight="bold" align="center" color="#fff">Title</headline>
        <paragraph size="s" color="<muted>" align="center" leading="relaxed">
          Brief description. Outcome-focused.
        </paragraph>
      </column>
    </row>

### Two-Column Content + List

    <row cols="2">
      <column>
        <headline size="l" weight="bold" color="#fff">Left<br/><span color="<a>">Side</span></headline>
        <paragraph size="m" color="<muted>" leading="relaxed">Supporting copy.</paragraph>
        <list icon="check" iconcolor="<accent>" size="m" color="<text>">
          <item>Specific benefit</item>
        </list>
      </column>
      <column>
        <headline size="l" weight="bold" color="#fff">Right<br/><span color="<b>">Side</span></headline>
        <list icon="bolt" iconcolor="<accent2>">
          <item>Another benefit</item>
        </list>
      </column>
    </row>

### Social Proof (testimonials)

  3 testimonials is ideal. 5+ feels desperate.
  Real names + headshots convert; generic "A. Customer" does not.

    <row cols="3">
      <column bg="<surface>" radius="lg" shadow="sm">
        <image prompt="headshot of professional in their 30s, warm smile, neutral bg" width="80px" radius="full" align="center"/>
        <paragraph size="s" color="<text>" align="center" leading="relaxed">
          "Specific result quote. Dollar amounts and timeframes beat vague claims."
        </paragraph>
        <paragraph size="s" color="<accent>" weight="bold" align="center">Name, Title</paragraph>
      </column>
    </row>

### Final CTA Section

  Contrasting background demands attention. Restate the core transformation.
  Mirror the hero CTA text for recognition.

    <section bg="linear-gradient(135deg, <primary> 0%, <secondary> 100%)" padding="100 40">
      <row>
        <column>
          <headline size="l" weight="black" align="center" color="#fff">
            Ready to <span color="<accent>">Start?</span>
          </headline>
          <paragraph size="m" color="rgba(255,255,255,0.75)" align="center">
            Brief restatement of the core promise.
          </paragraph>
          <button bg="#fff" color="<primary>" fontsize="18px" radius="full" align="center">
            Same CTA Text As Hero ->
          </button>
        </column>
      </row>
    </section>


================================================================================
## 14. DESIGN SYSTEM - IMAGE STRATEGY
================================================================================

  3-5+ images per page minimum for high-converting pages.
  Hero sections MUST include visual content.

### AI Prompt Quality

  WEAK prompts produce generic stock imagery. Be specific:

  Include: subject + lighting + emotion + environment + style

  Examples:
    WEAK:   "businesswoman smiling"
    STRONG: "professional woman in late 40s, warm confident smile,
             soft natural window light, modern home office with plants,
             editorial photography style, shallow depth of field"

    WEAK:   "abstract background"
    STRONG: "dark navy abstract geometric mesh gradient, subtle electric blue glow
             at edges, minimal, premium tech aesthetic, 16:9"

### Image Style Consistency

  If you're using multiple AI images, write a style guide for ALL of them:
  "All images: editorial photography, warm natural light, professional subjects
  aged 35-55, modern workspace environments"

  Then apply consistently to every <image prompt="..."/>.

### AI Image Limitations

  - Each AI image adds ~3-4 seconds to page load
  - Max 4 AI images per page (12-15s total is acceptable; 5+ = too slow)
  - Use bgprompt on sections sparingly (also AI-generated, same time cost)


================================================================================
## 15. DESIGN SYSTEM - INDUSTRY TEMPLATES
================================================================================

  Canonical section patterns per industry.

  coach_consultant
    Description:   Coaching programs, consulting, personal development
    Best styles:   modern_tech, luxury_premium, warm_inviting
    Section flow:  hero -> transformation -> credibility -> cta
    Key elements:  testimonials, credentials, transformation story
    Layouts:       hero(centered) -> 2-col(text+list) -> centered -> centered(input+button)

  saas_product
    Description:   Software, apps, digital tools
    Best styles:   modern_tech, minimalist_clean
    Section flow:  hero -> features -> pricing -> faq
    Key elements:  feature grid, pricing table, demo video
    Layouts:       hero(2-col: text+image) -> 3-col(feature cards) -> 3-col(pricing) -> accordion

  ecommerce
    Description:   Physical products, retail, D2C
    Best styles:   minimalist_clean, bold_striking, playful_creative
    Section flow:  hero -> products -> trust -> cta
    Key elements:  product images, reviews, trust badges
    Layouts:       hero(centered) -> 3-col(product cards) -> centered(list) -> centered(button)

  health_wellness
    Description:   Fitness, nutrition, mental health, wellness
    Best styles:   warm_inviting, minimalist_clean
    Section flow:  hero -> benefits -> proof -> cta
    Key elements:  before/after, testimonials, health benefits
    Layouts:       hero(centered) -> single-col(list) -> 2-col(text+image) -> centered(input+button)

  course_creator
    Description:   Online courses, digital education, training
    Best styles:   modern_tech, luxury_premium, playful_creative
    Section flow:  hero -> curriculum -> instructor -> cta
    Key elements:  module list, instructor bio, testimonials
    Layouts:       hero(centered) -> single-col(accordion) -> 2-col(image+text) -> centered

  local_business
    Description:   Local services, restaurants, professional services
    Best styles:   warm_inviting, modern_tech, minimalist_clean
    Section flow:  hero -> services -> about -> contact
    Key elements:  service list, location, contact form
    Layouts:       hero(centered) -> 3-col -> 2-col(image+text) -> centered(inputs+button)

  webinar
    Description:   Live events, workshops, masterclasses, webinars
    Best styles:   modern_tech, bold_striking, playful_creative
    Section flow:  hero -> learn -> host -> cta
    Key elements:  date/time countdown, learning points, host bio
    Layouts:       hero(centered with countdown) -> single-col(list) -> 2-col(image+text) -> centered

  optin
    Description:   Email capture, lead magnets, free downloads
    Best styles:   modern_tech, playful_creative, bold_striking
    Section flow:  hero -> benefits -> cta
    Key elements:  value proposition, benefit list, email form
    Layouts:       hero(centered: headline+input+button) -> single-col(list) -> centered


================================================================================
## 16. WHAT NOT TO DO
================================================================================

  Architecture:
  x Content directly in <section> or <row> - always wrap in <column>
  x <column> inside <section> without a <row> between them
  x <row> inside another <row>
  x padding attribute on <column> elements

  Colors:
  x rgba() in any color attribute - hex only
  x Named CSS colors ("red", "blue") - hex only
  x More than one accent color family per page (pick one, use it consistently)

  Typography:
  x Same font for headline AND body - always pair different families
  x Same visual weight for different hierarchy levels
  x fontsize without unit - "72" fails, "72px" works

  Layout:
  x <flex> for vertical layouts (flex direction="column" is not supported)
  x cols > 4 for content columns (layout grids can use more; content max is 4)
  x Padding "120" on mobile (set padding-mobile="60 20" for mobile comfort)

  Performance:
  x More than 4 AI-generated images per page (triggers 12-15s+ load time)
  x bgprompt on every section (1-2 max)

  Conversion:
  x Two CTAs in one section - one action, one button
  x Vague copy ("great results") - use specifics ("$47K in 30 days")
  x Hero headline under 72px - if it doesn't feel massive, it's too small
  x Padding under 80px on hero sections - cramped heroes lose trust
  x More than 5 testimonials - 3 is ideal

  customhtml:
  x Using customhtml for anything the DSL can handle (text, buttons, lists, cards)


================================================================================
## 17. COMPLETE EXAMPLES
================================================================================

  Four worked examples covering different industries, styles, and DSL patterns.
  Copy -> adapt -> POST through the consuming API (see Pages Skill or Blogs Skill).

### Example A: SaaS Landing Page (modern_tech)

  Industry: saas_product | Style: modern_tech
  Palette: hero #030712, content #0A0A14, accent #7C3AED/#A78BFA, muted #64748B
  Fonts: Poppins (headlines) + Inter (body)
  Patterns: gradient bg, stats strip, 3-col feature cards, 2-col list, final CTA

<section bg="linear-gradient(160deg, #030712 0%, #0F0826 50%, #030712 100%)" padding="120 40">
  <row>
    <column>
      <paragraph size="s" color="#7C3AED" weight="bold" align="center">* YOUR BRAND / TAGLINE *</paragraph>
      <headline fontsize="88px" weight="black" align="center" color="#FFFFFF" font="Poppins">
        The Tool That<br/><span color="#A78BFA">Pays For Itself.</span>
      </headline>
      <paragraph size="l" color="#94A3B8" align="center" leading="relaxed">
        One sentence that states the core value proposition.<br/>
        Second sentence with a specific outcome or number.
      </paragraph>
      <button bg="linear-gradient(135deg, #7C3AED 0%, #2563EB 100%)"
              color="#FFFFFF" font="Poppins" fontsize="18px" radius="full" align="center">
        Start Free Trial ->
      </button>
    </column>
  </row>
</section>

<section bg="#0A0A14" padding="0 0">
  <row cols="4">
    <column>
      <headline fontsize="52px" weight="black" align="center" color="#A78BFA" font="Poppins">10K+</headline>
      <paragraph size="s" color="#475569" align="center" weight="semibold">Active Users</paragraph>
    </column>
    <column>
      <headline fontsize="52px" weight="black" align="center" color="#34D399" font="Poppins">99.9%</headline>
      <paragraph size="s" color="#475569" align="center" weight="semibold">Uptime</paragraph>
    </column>
    <column>
      <headline fontsize="52px" weight="black" align="center" color="#60A5FA" font="Poppins">$2M</headline>
      <paragraph size="s" color="#475569" align="center" weight="semibold">Revenue Tracked</paragraph>
    </column>
    <column>
      <headline fontsize="52px" weight="black" align="center" color="#F472B6" font="Poppins">24/7</headline>
      <paragraph size="s" color="#475569" align="center" weight="semibold">Support</paragraph>
    </column>
  </row>
</section>

<section bg="#080C18" padding="100 40">
  <row>
    <column>
      <headline size="l" weight="bold" align="center" color="#FFFFFF" font="Poppins">
        Why Teams <span color="#A78BFA">Choose Us</span>
      </headline>
      <paragraph size="m" color="#475569" align="center">Three things that actually matter.</paragraph>
    </column>
  </row>
  <row cols="3">
    <column bg="#0f172a" radius="lg" shadow="md">
      <headline size="xl" align="center"></headline>
      <headline size="s" weight="bold" align="center" color="#FFFFFF" font="Poppins">Blazing Fast</headline>
      <paragraph size="s" color="#64748B" align="center" leading="relaxed">
        Up and running in under 5 minutes. No engineering required.
      </paragraph>
    </column>
    <column bg="#0f172a" radius="lg" shadow="md">
      <headline size="xl" align="center"></headline>
      <headline size="s" weight="bold" align="center" color="#FFFFFF" font="Poppins">Enterprise Secure</headline>
      <paragraph size="s" color="#64748B" align="center" leading="relaxed">
        SOC 2 compliant. Your data never leaves your workspace.
      </paragraph>
    </column>
    <column bg="#0f172a" radius="lg" shadow="md">
      <headline size="xl" align="center"></headline>
      <headline size="s" weight="bold" align="center" color="#FFFFFF" font="Poppins">Built to Scale</headline>
      <paragraph size="s" color="#64748B" align="center" leading="relaxed">
        From 10 users to 100,000. Same performance, same price per seat.
      </paragraph>
    </column>
  </row>
</section>

<section bg="#0D0D1F" padding="100 40">
  <row cols="2">
    <column>
      <headline size="l" weight="bold" color="#FFFFFF" font="Poppins">
        Everything<br/><span color="#A78BFA">You Need</span>
      </headline>
      <paragraph size="m" color="#64748B" leading="relaxed">
        All the tools in one place. No duct tape required.
      </paragraph>
      <list icon="check" iconcolor="#A78BFA" size="m" color="#CBD5E1" font="Inter">
        <item>Drag-and-drop page builder with 100+ templates</item>
        <item>Built-in email marketing and automation</item>
        <item>Native checkout and payment processing</item>
        <item>A/B testing on every page element</item>
      </list>
    </column>
    <column>
      <headline size="l" weight="bold" color="#FFFFFF" font="Poppins">
        Nothing<br/><span color="#34D399">To Worry About</span>
      </headline>
      <paragraph size="m" color="#64748B" leading="relaxed">
        We handle the infrastructure so you can focus on growth.
      </paragraph>
      <list icon="bolt" iconcolor="#34D399" size="m" color="#CBD5E1" font="Inter">
        <item>Unlimited bandwidth on all plans</item>
        <item>Free SSL certificate and CDN included</item>
        <item>Cancel any time - no contracts</item>
        <item>30-day money-back guarantee, no questions</item>
      </list>
    </column>
  </row>
</section>

<section bg="linear-gradient(135deg, #1E0845 0%, #0C1B4D 100%)" padding="100 40">
  <row>
    <column>
      <headline size="l" weight="black" align="center" color="#FFFFFF" font="Poppins">
        Ready to Start?
      </headline>
      <paragraph size="m" color="#A78BFA" align="center" leading="relaxed">
        Join 10,000+ teams already using the platform.
      </paragraph>
      <button bg="#FFFFFF" color="#1E0845" font="Poppins" fontsize="18px" radius="full" align="center">
        Start Free Trial ->
      </button>
    </column>
  </row>
</section>

### Example B: Health & Wellness Opt-In (warm_inviting)

  Industry: health_wellness | Style: warm_inviting
  Palette: hero #065f46, content #fffbeb, accent #10b981, muted #78716c
  Fonts: Libre Baskerville (headlines) + Roboto (body)
  Patterns: <input>, <list>, light backgrounds, warm palette

<section bg="#065f46" padding="100 40">
  <row>
    <column>
      <paragraph size="s" color="#10b981" weight="bold" align="center" font="Roboto" transform="uppercase" spacing="wider">FREE WELLNESS GUIDE</paragraph>
      <headline fontsize="72px" weight="bold" align="center" color="#ffffff" font="Libre Baskerville">
        Reclaim Your <span color="#a7f3d0">Energy</span><br/>in 21 Days
      </headline>
      <paragraph size="m" color="#d1fae5" align="center" leading="relaxed" font="Roboto">
        A science-backed daily plan that fits into your morning routine.<br/>
        Join 5,000+ people who wake up feeling alive again.
      </paragraph>
      <input label="Email Address" type="email" placeholder="you@example.com" required="true" width="100%" radius="md"/>
      <button bg="#10b981" color="#ffffff" font="Roboto" fontsize="18px" radius="xl" align="center">
        Send Me the Free Guide ->
      </button>
      <paragraph size="s" color="#6ee7b7" align="center" weight="medium">No spam. Unsubscribe any time.</paragraph>
    </column>
  </row>
</section>

<section bg="#fffbeb" padding="80 40">
  <row>
    <column>
      <headline size="l" weight="bold" align="center" color="#1c1917" font="Libre Baskerville">
        What You'll <span color="#065f46">Discover</span>
      </headline>
      <paragraph size="m" color="#78716c" align="center" font="Roboto">Three pillars that change everything.</paragraph>
    </column>
  </row>
  <row>
    <column>
      <list icon="check" iconcolor="#10b981" size="m" color="#1c1917" font="Roboto">
        <item>The 5-minute morning reset that eliminates brain fog by 9 AM</item>
        <item>Why 80% of fatigue starts in the gut - and the 3 foods that fix it</item>
        <item>A simple evening wind-down protocol for deep, restorative sleep</item>
        <item>How to maintain energy without caffeine crashes or sugar spikes</item>
      </list>
    </column>
  </row>
</section>

<section bg="linear-gradient(135deg, #065f46 0%, #064e3b 100%)" padding="80 40">
  <row>
    <column>
      <headline size="l" weight="bold" align="center" color="#ffffff" font="Libre Baskerville">
        Start Feeling <span color="#a7f3d0">Different</span> Tomorrow
      </headline>
      <paragraph size="m" color="#d1fae5" align="center" leading="relaxed" font="Roboto">
        The guide is free. The results are priceless.
      </paragraph>
      <input label="Email Address" type="email" placeholder="you@example.com" required="true" width="100%" radius="md"/>
      <button bg="#10b981" color="#ffffff" font="Roboto" fontsize="18px" radius="xl" align="center">
        Send Me the Free Guide ->
      </button>
    </column>
  </row>
</section>

### Example C: Course Creator Sales Page (luxury_premium)

  Industry: course_creator | Style: luxury_premium
  Palette: hero #1a1a2e, content #faf5ef, accent #d4af37, muted #8b7355
  Fonts: Playfair Display (headlines) + Lato (body)
  Patterns: <accordion>, <image prompt="">, bgprompt, 2-column layout

<section bg="#1a1a2e" bgprompt="dark elegant marble texture with gold flecks, subtle luxury feel, low light" padding="120 40">
  <row>
    <column>
      <paragraph size="s" color="#d4af37" weight="bold" align="center" font="Lato" transform="uppercase" spacing="widest">MASTER PROGRAMME</paragraph>
      <headline fontsize="80px" weight="normal" align="center" color="#faf5ef" font="Playfair Display">
        The Art of<br/><span color="#d4af37">Persuasive Writing</span>
      </headline>
      <paragraph size="l" color="#8b7355" align="center" leading="relaxed" font="Lato" weight="light">
        12 weeks. 47 lessons. One transformation.<br/>
        Write copy that commands attention and closes deals.
      </paragraph>
      <button bg="linear-gradient(135deg, #d4af37 0%, #b8962e 100%)" color="#1a1a2e" font="Lato" fontsize="18px" radius="md" align="center">
        Enrol Now - $497 ->
      </button>
    </column>
  </row>
</section>

<section bg="#faf5ef" padding="100 40">
  <row>
    <column>
      <headline size="l" weight="normal" align="center" color="#1a1a2e" font="Playfair Display">
        What You'll <span color="#d4af37">Master</span>
      </headline>
      <paragraph size="m" color="#8b7355" align="center" font="Lato" weight="light">The complete curriculum, module by module.</paragraph>
    </column>
  </row>
  <row>
    <column>
      <accordion bg="#ffffff" textcolor="#1a1a2e">
        <item title="Module 1 - The Psychology of Persuasion">Understand the cognitive triggers that make readers say yes. Covers anchoring, social proof, loss aversion, and the curiosity gap.</item>
        <item title="Module 2 - Headlines That Stop the Scroll">Craft headlines using the 4-U framework. 15 proven templates with real-world breakdowns from $1M+ campaigns.</item>
        <item title="Module 3 - Story-Driven Sales Pages">Structure long-form copy using the hero's journey. Turn features into transformation narratives that sell.</item>
        <item title="Module 4 - Email Sequences That Convert">Build 7-figure welcome, launch, and evergreen sequences. Includes swipe files and automation blueprints.</item>
      </accordion>
    </column>
  </row>
</section>

<section bg="#1a1a2e" padding="100 40">
  <row cols="2">
    <column>
      <image prompt="distinguished male writing instructor in his 50s, silver hair, warm confident expression, seated at mahogany desk with leather-bound books, soft golden lamp light, editorial portrait style, shallow depth of field" width="100%" radius="md"/>
    </column>
    <column>
      <paragraph size="s" color="#d4af37" weight="bold" font="Lato" transform="uppercase" spacing="wider">YOUR INSTRUCTOR</paragraph>
      <headline size="l" weight="normal" color="#faf5ef" font="Playfair Display">
        James <span color="#d4af37">Ashford</span>
      </headline>
      <paragraph size="m" color="#8b7355" leading="relaxed" font="Lato" weight="light">
        20 years in direct-response copywriting. $47M in client revenue generated.
        Former head of copy at two Fortune 500 agencies. Published author of
        "Words That Sell" - a Wall Street Journal bestseller.
      </paragraph>
      <list icon="star" iconcolor="#d4af37" size="m" color="#faf5ef" font="Lato">
        <item>Wrote copy for Apple, Nike, and Shopify</item>
        <item>Trained 3,000+ students across 40 countries</item>
        <item>Average student ROI: 11x within 90 days</item>
      </list>
    </column>
  </row>
</section>

<section bg="#faf5ef" padding="100 40">
  <row>
    <column>
      <headline size="l" weight="normal" align="center" color="#1a1a2e" font="Playfair Display">
        What Students <span color="#d4af37">Say</span>
      </headline>
    </column>
  </row>
  <row cols="3">
    <column bg="#ffffff" radius="md" shadow="sm">
      <paragraph size="s" color="#1a1a2e" align="center" leading="relaxed" font="Lato">
        "I rewrote my sales page after Module 3 and tripled my conversion rate in two weeks. This course paid for itself on day one."
      </paragraph>
      <paragraph size="s" color="#d4af37" weight="bold" align="center" font="Lato">Sarah Chen, Course Creator</paragraph>
    </column>
    <column bg="#ffffff" radius="md" shadow="sm">
      <paragraph size="s" color="#1a1a2e" align="center" leading="relaxed" font="Lato">
        "The email sequence templates alone are worth 10x the price. My open rates went from 18% to 41%."
      </paragraph>
      <paragraph size="s" color="#d4af37" weight="bold" align="center" font="Lato">Marcus Rivera, Agency Owner</paragraph>
    </column>
    <column bg="#ffffff" radius="md" shadow="sm">
      <paragraph size="s" color="#1a1a2e" align="center" leading="relaxed" font="Lato">
        "James breaks down the 'why' behind every technique. I finally understand what makes great copy work."
      </paragraph>
      <paragraph size="s" color="#d4af37" weight="bold" align="center" font="Lato">Priya Patel, Freelance Writer</paragraph>
    </column>
  </row>
</section>

<section bg="linear-gradient(135deg, #1a1a2e 0%, #16213e 100%)" padding="100 40">
  <row>
    <column>
      <headline size="l" weight="normal" align="center" color="#faf5ef" font="Playfair Display">
        Your Words Are <span color="#d4af37">Worth More</span>
      </headline>
      <paragraph size="m" color="#8b7355" align="center" leading="relaxed" font="Lato" weight="light">
        Enrolment closes Friday. 30-day money-back guarantee.
      </paragraph>
      <button bg="linear-gradient(135deg, #d4af37 0%, #b8962e 100%)" color="#1a1a2e" font="Lato" fontsize="18px" radius="md" align="center">
        Enrol Now - $497 ->
      </button>
    </column>
  </row>
</section>

### Example D: E-Commerce Product Page (bold_striking)

  Industry: ecommerce | Style: bold_striking
  Palette: hero #000000, content #ffffff, accent #ef4444, muted #4b5563
  Fonts: Anton (headlines) + Roboto (body)
  Patterns: <image src="">, <flex>, radius="none" (sharp edges), high-contrast

<section bg="#000000" padding="120 40">
  <row cols="2">
    <column>
      <paragraph size="s" color="#ef4444" weight="bold" font="Roboto" transform="uppercase" spacing="widest">NEW RELEASE</paragraph>
      <headline fontsize="88px" weight="normal" color="#ffffff" font="Anton" transform="uppercase">
        APEX<br/>RUNNER X1
      </headline>
      <paragraph size="m" color="#9ca3af" leading="relaxed" font="Roboto">
        Engineered for the relentless. Carbon-plate midsole.<br/>
        4mm drop. 6.2 oz. The fastest shoe we've ever built.
      </paragraph>
      <flex direction="row" gap="16" wrap="nowrap">
        <button bg="#ef4444" color="#ffffff" font="Roboto" fontsize="18px" radius="none" align="center">
          Buy Now - $189
        </button>
        <button bg="transparent" color="#ef4444" font="Roboto" fontsize="18px" radius="none" border="medium" align="center">
          See It In Action
        </button>
      </flex>
    </column>
    <column>
      <image src="https://images.unsplash.com/photo-1542291026-7eec264c27ff?w=800" width="100%" radius="none" alt="Apex Runner X1 - angular product shot on black background"/>
    </column>
  </row>
</section>

<section bg="#ffffff" padding="80 40">
  <row>
    <column>
      <headline size="l" weight="normal" align="center" color="#000000" font="Anton" transform="uppercase">
        Built to <span color="#ef4444">Dominate</span>
      </headline>
    </column>
  </row>
  <row cols="3">
    <column bg="#f9fafb" radius="none" shadow="lg">
      <headline size="xl" align="center"></headline>
      <headline size="s" weight="normal" align="center" color="#000000" font="Anton" transform="uppercase">Carbon Plate</headline>
      <paragraph size="s" color="#4b5563" align="center" leading="relaxed" font="Roboto">
        Full-length carbon-fibre plate returns 87% energy on each stride.
      </paragraph>
    </column>
    <column bg="#f9fafb" radius="none" shadow="lg">
      <headline size="xl" align="center"></headline>
      <headline size="s" weight="normal" align="center" color="#000000" font="Anton" transform="uppercase">Featherweight</headline>
      <paragraph size="s" color="#4b5563" align="center" leading="relaxed" font="Roboto">
        6.2 oz total weight. 23% lighter than our previous fastest model.
      </paragraph>
    </column>
    <column bg="#f9fafb" radius="none" shadow="lg">
      <headline size="xl" align="center"></headline>
      <headline size="s" weight="normal" align="center" color="#000000" font="Anton" transform="uppercase">Precision Fit</headline>
      <paragraph size="s" color="#4b5563" align="center" leading="relaxed" font="Roboto">
        3D-knit upper locks your foot in place. Zero wasted movement.
      </paragraph>
    </column>
  </row>
</section>

<section bg="#000000" padding="80 40">
  <row>
    <column>
      <headline size="l" weight="normal" align="center" color="#ffffff" font="Anton" transform="uppercase">
        Trusted by <span color="#ef4444">Champions</span>
      </headline>
      <paragraph size="m" color="#4b5563" align="center" font="Roboto">Worn at 14 world championship events. 9 podium finishes.</paragraph>
    </column>
  </row>
  <row cols="4">
    <column>
      <headline fontsize="52px" weight="normal" align="center" color="#ef4444" font="Anton">14</headline>
      <paragraph size="s" color="#9ca3af" align="center" weight="semibold" font="Roboto">Championships</paragraph>
    </column>
    <column>
      <headline fontsize="52px" weight="normal" align="center" color="#ffffff" font="Anton">9</headline>
      <paragraph size="s" color="#9ca3af" align="center" weight="semibold" font="Roboto">Podium Finishes</paragraph>
    </column>
    <column>
      <headline fontsize="52px" weight="normal" align="center" color="#ef4444" font="Anton">2:01</headline>
      <paragraph size="s" color="#9ca3af" align="center" weight="semibold" font="Roboto">Marathon PR</paragraph>
    </column>
    <column>
      <headline fontsize="52px" weight="normal" align="center" color="#ffffff" font="Anton">50K+</headline>
      <paragraph size="s" color="#9ca3af" align="center" weight="semibold" font="Roboto">Pairs Sold</paragraph>
    </column>
  </row>
</section>

<section bg="#ef4444" padding="80 40">
  <row>
    <column>
      <headline size="l" weight="normal" align="center" color="#ffffff" font="Anton" transform="uppercase">
        Don't Wait. Dominate.
      </headline>
      <paragraph size="m" color="#fecaca" align="center" font="Roboto">
        Limited first run. Ships in 2-4 business days.
      </paragraph>
      <button bg="#000000" color="#ffffff" font="Roboto" fontsize="18px" radius="none" align="center">
        Buy Now - $189
      </button>
    </column>
  </row>
</section>


================================================================================
## 18. ERROR REFERENCE
================================================================================

  HTTP 400 - invalid top-level DSL element:
    { "error": "Bad request: Markup is not valid PML: unknown top-level element(s) <foo>. ... (line 3)" }

  HTTP 400 - non-PML input (plain text, raw HTML):
    { "error": "Bad request: Markup is not valid PML: no PML elements found. ..." }

  HTTP 400 - nesting violation:
    { "error": "Bad request: <column> element must be inside a <row>. ..." }

  HTTP-level errors (401 missing token, 404 not found, etc.) are returned by the
  consuming API - see the Pages Skill or Blogs Skill for those.


================================================================================
## 19. PRE-POST CHECKLIST
================================================================================

  Structure:
  [ ] Every <section> has at least one <row>
  [ ] Every <row> has at least one <column>
  [ ] All content elements are inside a <column>
  [ ] No padding attribute on <column> elements
  [ ] No <row> nested inside another <row>

  Colors:
  [ ] All colors are #hex (no rgba, no named colors)
  [ ] Gradient presets used by name OR full CSS gradient string

  Typography:
  [ ] fontsize values include unit: "72px" not "72"
  [ ] Font names are exact Google Font names
  [ ] Headline font != body font (different families paired)

  Images:
  [ ] AI image prompts include subject + lighting + emotion + environment
  [ ] <=4 AI-generated images per page (performance)
  [ ] bgprompt used on <=2 sections

  Conversion:
  [ ] Each section has exactly one CTA button
  [ ] Hero headline fontsize >= 72px
  [ ] Hero section padding >= 100px

  <flex>:
  [ ] Only used with direction="row" (horizontal layouts only)


================================================================================
# END OF page-markup/skill.md
# ClickFunnels 2.0 Page Markup Language Reference v2.0
================================================================================
