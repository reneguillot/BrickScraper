# BrickScraper
Date: January 7, 2024

## Introduction
We want to periodically (e.g. daily or weekly) scrape the full contents of the https://www.brickwatch.net ("BW") and convert it into an Excel-friendly output format (CSV files). The contents of this website primarily consists of:

* A full list of (current) LEGO sets sold. At time of writing, BrickWatch contains 5.530+ LEGO sets
* For each LEGO set: a list of registered resellers, including end consumer prices

The (custom) solution envisioned to scrape BW is dubbed "BrickScraper" (a codename for now). The solution is periodically run in the cloud (AWS - Amazon Web Services), without the need to manage servers/hardware ourselves. After collecting relevant data and constructing the output CSV file, ideally it is sent to a predefined e-mail address (as an attachment). The e-mail recipient can then manually process the CSV file as considered useful.

## BrickWatch (BW) Website Analysis

### Index Data
The full LEGO sets list ("index") consists of 27 distinct URLs, one for each alphabet character, and an additional 0 (zero) for numbers:

* https://www.brickwatch.net/nl-NL/sets/char/0?order=set
* https://www.brickwatch.net/nl-NL/sets/char/A?order=set
* https://www.brickwatch.net/nl-NL/sets/char/B?order=set
* https://www.brickwatch.net/nl-NL/sets/char/C?order=set
* ...
* https://www.brickwatch.net/nl-NL/sets/char/Z?order=set

Note that the above **order** URL query string parameter is optional, but at least this sorts the data by LEGO set number. The index is the starting point for retrieving individual LEGO sets, one by one.

BW doesn't seem to be protected by *User-Agent* string verification. In other words: it doesn't seem to care whether the URL is called within a web browser (which would be considered conventional use), Postman or any automated fashion. *User-Agent* request header can be omitted (tested in PostMan), which is promising for automatic processing.

After fetching the above URLs, the following regular expression on the HTML code retrieves links to all LEGO sets starting with that character/number:

<a href='(.+)' name='(.+)'>

Group 1 returns the direct deeplink into the LEGO set on the website; group 2 returns the LEGO set number which happens also to be part of the URL. Some examples:

<a href='/nl-NL/set/5007962/Baby-Dragon.html' name='5007962'>
<a href='/nl-NL/set/10300/Back-to-the-Future-Time-Machine.html' name='10300'>
<a href='/nl-NL/set/60333/Bathtub-Stunt-Bike.html' name='60333'>
<a href='/nl-NL/set/5005580/LEGO-Banana-Guy-Luggage-Tag.html' name='5005580'>

### Individual LEGO Set Data
Each of the LEGO set deeplinks in the index must be retrieved individually. Again (like the index pages), *User-Agent* doesn't seem to be actively checked by BW, so Postman and/or automated processing seems possible.
Optionally, the URL could be suffixed with the following query string parameter, sorting the resellers by ascending price:

```
?order=p#setprices
```

Description:

```html
<div class="float-start">
	<h2 class="h4 fw-bold">Back to the Future tijdmachine (10300)</h2>
</div>
```

Notes:
* DIV (float-start) and H2 (h4 fw-bold) seems to be found once.
* LEGO set number (10300) must be removed from the name

Prices:

```html
<a name="setprices"></a>
...
<table class="table align-middle table-xs-sm mb-0 table-hover">
<thead>
...
</thead>
<tbody>
```

Notes:
* Table with this class is found once, skip <thead> and process <tbody> (<tr> elements), below:

```html
<tr style='height: 40px'>
  <td class='text-center'><a href='/nl-NL/go/33/10300'
      tag='set_list_sitelogo-1' rel='nofollow'
      target='_blank'><img data-src='https://cdn.kiobi.com/images/sites/33.png' src='https://cdn.kiobi.com/images/shared/spinner.gif' class='lazyload' title="Mister Bricks" style='width: 100px; height: 25px;'></a>
  </td>
  <td class="d-none d-sm-table-cell darklink-no"><a
      href='/nl-NL/go/33/10300' rel='nofollow' class='fw-normal'
      tag='set_list_description-1' target='_blank'>
      <div
        style="position:relative;width:100%;margin-top: -12px;">
        <span style="position:absolute;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;width:100%;min-width:0;top:0;left:0;">LEGO 10300 Back to the Future tijdmachine</span>
      </div>
    </a></td>
  <td class='darklink-no text-end'>
    <nobr><a href='/nl-NL/site/33/'><img src='https://cdn.kiobi.com/images/brickwatch/r1.png' width=18 height=18 alt=''> 4.7</a>
    </nobr>
  </td>
  <td class='d-none d-sm-table-cell text-start fw-semibold py-0'>
    <nobr>
      <nobr>
        <font color=orange>
          <b title='Weer beschikbaar'>&orarr;</b></font>
      </nobr>
      <div class='small fw-normal' style='margin-top: -4px;'>
        <nobr class='small'>Jan 02, 2024</nobr>
      </div>
    </nobr>
  </td>
  <td class='text-start'
    title='Laatste controle: 2024-01-07 19:42:25GMT'>
    <nobr><a href='/nl-NL/go/33/10300' rel='nofollow'
        tag='set_list_price-1' target='_blank'>&euro; 169,00</a>
    </nobr>
  </td>
  <td class='text-start text-nowrap'><a href='/nl-NL/go/33/10300'
      rel='nofollow' tag='set_list_deliveredprice-1'
      style='text-decoration: none; color: black;'
      target='_blank'>&euro; 169,00</a></td>
  <td class='d-sm-none text-start fw-semibold py-0'>
    <nobr>
      <nobr>
        <font color=orange>
          <b title='Weer beschikbaar'>&orarr;</b></font>
      </nobr>
      <div class='small fw-normal' style='margin-top: -4px;'>
        <nobr class='small'>Jan 02, 2024</nobr>
      </div>
    </nobr>
  </td>
</tr>
```

Notes:
* Even though <tr> seems to contain 7 <td> elements, only the first 6 are visually displayed on the BW website. Therefore, discard the 7th <td> element.
* The <a> element within the first <td> element seems to contain the only reseller reference, within the embedded <img> element. Look for **title** attribute.
* The <a> element within the first <td> element contains a referral URL directly into the respective reseller.
* History / Last Change: several <td> element types seems to be used:

<u>Type 1 (font color=red: more expensive with last change date):</u>

```html
<td class='d-none d-sm-table-cell text-start fw-semibold py-0'>
  <nobr>
    <nobr>
      <font color=red>
        <b>&uarr;</b><small title='Was &euro; 169,99'> &euro; 30,00</small>
      </font>
    </nobr>
    <div class='small fw-normal' style='margin-top: -4px;'>
      <nobr class='small'>Aug 31, 2022</nobr>
    </div>
  </nobr>
</td>
```

<u>Type 2 (font color=orange: available again with last change date)</u>:

```html
<td class='d-none d-sm-table-cell text-start fw-semibold py-0'>
  <nobr>
    <nobr>
      <font color=orange>
        <b title='Weer beschikbaar'>&orarr;</b></font>
    </nobr>
    <div class='small fw-normal' style='margin-top: -4px;'>
      <nobr class='small'>Jan 06, 2024</nobr>
    </div>
  </nobr>
</td>
```

<u>Type 3 (font color=green; cheaper with last change date):</u>

```html
<td class='d-none d-sm-table-cell text-start fw-semibold py-0'>
  <nobr>
    <nobr>
      <font color='green'>
        <b>&darr;</b><small title='Was &euro; 50,99'> &euro; 3,00</small>
      </font>
    </nobr>
    <div class='small fw-normal' style='margin-top: -4px;'>
      <nobr class='small'>Dec 28, 2023</nobr>
    </div>
  </nobr>
</td>
```

<u>Type 3 (no font, single <nobr> within <td>: no historical data available/allowed):</u>

```html
<td class='d-sm-none text-start fw-semibold py-0'>
  <nobr>
    <small><span style="opacity:0.7;" data-bs-toggle="tooltip" data-placement="top" title="Van sommige verkopers mogen we geen oude prijzen laten zien">&#10033;</span></small>
  </nobr>
</td>
```

<u>Type 4 (font color=blue, unchanged with last change date):</u>

```html
<td class='d-none d-sm-table-cell text-start fw-semibold py-0'>
  <nobr>
    <font color=blue>-</font>
    <div class='small fw-normal' style='margin-top: -4px;'>
      <nobr class='small'>May 23, 2023</nobr>
    </div>
  </nobr>
</td>
```

* Price: 5th <td> element with <nobr> and <a> (nested) elements within. Anchor holds price including currency symbol in HTML special character notation.
* Price including shipping: 6td> element with <a> (nested) element within. Anchor holds price including currency symbol in HTML special character notation.

## Output CSV Structure
In general, the CSV file format is designed to be as Excel friendly as possible. To simplify use of the data in Excel (opening up the file without triggering the Import Wizard), a .CSV file extension is used with a comma as field separator. Furthermore, specific data is formatted accordingly, preventing (automatic) misinterpretation by Excel:

* CSV files are always saved in UTF-8 encoding.
* Single dates (without time portion) are hardcoded in xx-xx-xxxx format.
* Field values which may use comma, should be enclosed in double quotes, explicitly converting the data as text data.

The following fields are expected:
* LEGO Set ID
* Description
* Price position in source data. Given the reseller data is sorted by price, position '1' would indicate the cheapest option
* Price
* History (if data available)
* Last changed (if data available)
* Reseller referral URL (into BW website, which should in turn redirect to the reseller)

For each reseller found on the LEGO set page, the row is duplicated on the first two columns (Set ID and Description).

## Anticipated AWS Architecture and Development Techniques
At the moment, the following (AWS) architecture is expected:
* ECS Fargate. Considering BW currently contains over 5530 individual LEGO sets, an AWS Lambda function is considered less useful here. Primary concerns (for now) is that a Lambda function must finish within the theoretical maximum of 15 minutes. This timeframe would probably already be too short to run about 5600 RESTful GET requests to the BW website (retrieving all HTML content of the index pages and all individual LEGO set pages)
* AWS Timed Event launching the ECS Fargate process, periodically (to be determined: daily/weekly)

At the moment, following development techniques are considered:
* NodeJS script. Python may be a good alternative, but Fargate supports both. NodeJS may have some further advantages
* ...

### Anticipated BrickScraper Configuration
The following (custom) configuration is expected for BrickScraper:
* BW Root URL: expected to be fixed and set to 'https://www.brickwatch.net'.
* Language/Country code designator in the following syntax: **languagecode-countrycode**. Language code is 2 characters lowercase according to ISO 639-1; country code is 2 characters uppercase according to ISO 3166-1. It is expected that this language/country designator is used in the (initial) index data retrieval. After this, it is likely that all individual LEGO set references follow the language/country designator. For the moment, it is expected that this designator is set to 'nl-NL'.
* Throttling delay (milliseconds): Time between subsequent RESTful requests to the BW website. It is highly encouraged to implement some form of throttling delay, so that BW isn't bombarded by BrickScraper in a very short time period. Keeping a low profile is desired, avoiding any connectivity issue / client blacklisting by BW (if implemented at all, but better safe than sorry).
