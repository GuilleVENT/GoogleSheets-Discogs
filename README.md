## Google Sheet + Discogs = Your Virtual Analog Catalogue

All you will need will be a Google Sheet on your drive with the following template. 

It looks something like this 

| Album Cover | Artist | Title | Year Released | Label | Genre | Style | Year Pressed | Print Country | My Notes | Discogs Notes | Unique Identifier | URL (Discogs) | Discogs Release ID | other... |
| ----------- | ------ | ----- | ------------- | ----- | ----- | ----- | ------------ | ------------- | -------- | ------------- | ----------------- | ------------- | ------------------ | -------- |
|             |        |       |               |       |       |       |              |               |          |               |                   |               |                    |          |

The workflow of summing your record collection into a digital table should be as simple as possible. The hardest part of finding a record is finding it on Discogs, specially if it's an old release and you gotta tweak your eyes and put the light just at the right angle. So with this macro/bot/service, that is all you actually will have to do. 

First, do just that, find your version of your record on Discogs. 

Then copy the URL or the release ID (without your country-code) to the respective columns  "URL (Discogs)" or "Discogs Release ID". The macro will do the rest. 

But first, we need to register our little app in the Discogs API, so go [here](https://www.discogs.com/settings/developers) and register yourself. Then you have the options to create an application or use a temporal and personal token. You can choose what you prefer to do. If you decide to go with the application, remember that you won't need the secret key, that's secret shhh (it acts as a sort of password for your requests).

```javascript
const DISCOGS_TOKEN = "<your-discogs-token>"
```

```javascript
function extractReleaseId(url) {
	const regex = /https:\/\/www\.discogs\.com\/(?:[a-z]{2}\/)?release\/(\d+)/;
	const match = url.match(regex);
	return match ? match[1] : null;
}
```

```javascript
function fetchDiscogsData() {

	// Get the first row (header row)
	const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
	const headers = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0]; 
	  
	// Dynamically find column indices by header name
	const urlColumn = headers.indexOf("URL (Discogs)") + 1;
	const artistColumn = headers.indexOf("Artist") + 1;
	const albumColumn = headers.indexOf("Title") + 1;
	const labelColumn = headers.indexOf("Label") + 1;
	const yearPressedColumn = headers.indexOf("Year Pressed") + 1;
	const yearReleasedColumn = headers.indexOf("Year Released") + 1;
	// myNotesColumn in contrast to My Notes
	const discogsNotesColumn = headers.indexOf("Discogs Notes") + 1;
	const albumCoverColumn = headers.indexOf("Album Cover") + 1;
	const countryColumn = headers.indexOf("Country") + 1;
	const genresColumn = headers.indexOf("Genres") + 1;
	const stylesColumn = headers.indexOf("Styles") + 1;
	const lowestPriceColumn = headers.indexOf("Lowest Price Sold") + 1;
	//const priceLowColumn = headers.indexOf("Price Low") + 1;
	//const priceMedianColumn = headers.indexOf("Price Median") + 1;
	//const priceMaxColumn = headers.indexOf("Price Max") + 1;
	const numForSaleColumn = headers.indexOf("Num for Sale") + 1;
	const quantityColumn = headers.indexOf("Quantity") + 1;
	const uniqueIdentifierColumn = headers.indexOf("Unique Identifier") + 1;
	
	if (urlColumn === 0) { 
		throw new Error("The 'URL (Discogs)' column is missing in the sheet header!"); 
	}
	
	const lastRow = sheet.getLastRow();
	const dataRange = sheet.getRange(2, urlColumn, lastRow - 1); // Skip the header row
	const urls = dataRange.getValues();
	
	for (let i = 0; i < urls.length; i++) {
		const url = urls[i][0];
		if (!url) continue; // Skip empty rows
		
		const releaseId = extractReleaseId(url);
		if (!releaseId) {
			sheet.getRange(i + 2, urlColumn).setNote("Invalid URL");
			continue;
		}

	const apiUrl = `https://api.discogs.com/releases/${releaseId}`;
	
	const options = {
		method: "get",
		headers: {
			Authorization: `Discogs token=${DISCOGS_TOKEN}`,	
			},
		};

	try {

	const response = UrlFetchApp.fetch(apiUrl, options);
	const releaseData = JSON.parse(response.getContentText());
	
	// Extract unique identifiers
	let uniqueIdentifiers = "No identifiers available";
	if (releaseData.identifiers && releaseData.identifiers.length > 0) {
		uniqueIdentifiers = releaseData.identifiers.map((identifier) => {
			const type = identifier.type || "Unknown Type";
			const value = identifier.value || "Unknown Value";
			const description = identifier.description || "";
			return `${type}: ${value}${description ? ` (${description})` : ""}`;
			}).join("\n");
	
	}

	let yearReleased = "";
	const masterId = releaseData.master_id || null;
	if (masterId) {
		const masterApiUrl = `https://api.discogs.com/masters/${masterId}`;
		const masterResponse = UrlFetchApp.fetch(masterApiUrl, options);
		const masterData = JSON.parse(masterResponse.getContentText());
		yearReleased = masterData.year || ""; // Original release year
	}
	
	const artist = releaseData.artists ? releaseData.artists.map((a) => a.name).join(", ") : "";
	
	const album = releaseData.title || "";
	const label = releaseData.labels ? releaseData.labels.map((l) => l.name).join(", ") : "";
	const yearPressed = releaseData.year || "";
	
	const country = releaseData.country || "Unknown";
	const genres = releaseData.genres ? releaseData.genres.join(", ") : "";
	const styles = releaseData.styles ? releaseData.styles.join(", ") : "";
	const lowestPrice = releaseData.lowest_price || "N/A";
	//const priceLow = releaseData.community && releaseData.community.price ? releaseData.community.price.min : "N/A";
	//const priceMedian = releaseData.community && releaseData.community.price ? releaseData.community.price.median : "N/A";
	//const priceMax = releaseData.community && releaseData.community.price ? releaseData.community.price.max : "N/A";
	const numForSale = releaseData.num_for_sale || 0;
	const quantity = releaseData.formats && releaseData.formats.length > 0
	? releaseData.formats.map((format) => format.qty || "Unknown").join(", ")
	: "Unknown"; //BUGBUGBUGBUGBUG
	
	const albumCover = releaseData.images && releaseData.images.length > 0 ? releaseData.images[0].uri : "";

	let discogsNotes = releaseData.notes || "";
	
	// Limit Discogs Notes to 13 lines max
	const maxLines = 13;
	const lines = discogsNotes.split("\n");
	if (lines.length > maxLines) {
		discogsNotes = lines.slice(0, maxLines).join("\n") + "\n...";
	}
	// Limit Unique Identifiers to 13 lines max
	const lines2 = uniqueIdentifiers.split("\n");
	if (lines2.length > maxLines) {
		uniqueIdentifiers = lines2.slice(0, maxLines).join("\n") + "\n...";
	}

	// Populate data into corresponding columns
	if (artistColumn) sheet.getRange(i + 2, artistColumn).setValue(artist);
	if (albumColumn) sheet.getRange(i + 2, albumColumn).setValue(album);
	if (labelColumn) sheet.getRange(i + 2, labelColumn).setValue(label);
	if (yearPressedColumn) sheet.getRange(i + 2, yearPressedColumn).setValue(yearPressed);
	if (yearReleasedColumn) sheet.getRange(i + 2, yearReleasedColumn).setValue(yearReleased);
	if (countryColumn) sheet.getRange(i + 2, countryColumn).setValue(country);
	if (genresColumn) sheet.getRange(i + 2, genresColumn).setValue(genres);
	if (stylesColumn) sheet.getRange(i + 2, stylesColumn).setValue(styles);
	if (lowestPriceColumn) sheet.getRange(i + 2, lowestPriceColumn).setValue(lowestPrice);
	//if (priceLowColumn) sheet.getRange(i + 2, priceLowColumn).setValue(priceLow);
	//if (priceMedianColumn) sheet.getRange(i + 2, priceMedianColumn).setValue(priceMedian);
	//if (priceMaxColumn) sheet.getRange(i + 2, priceMaxColumn).setValue(priceMax);
	if (numForSaleColumn) sheet.getRange(i + 2, numForSaleColumn).setValue(numForSale);
	if (quantityColumn) sheet.getRange(i + 2, quantityColumn).setValue(quantity);
	if (discogsNotesColumn) sheet.getRange(i + 2, discogsNotesColumn).setValue(discogsNotes);
	if (uniqueIdentifierColumn) {
		sheet.getRange(i + 2,uniqueIdentifierColumn).setValue(uniqueIdentifiers);
	}
	
	if (albumCoverColumn) {
		const size = 200;
		sheet.setColumnWidth(albumCoverColumn, size);
		sheet.setRowHeight(i + 2, size);
		if (albumCover) {
			sheet.getRange(i + 2,albumCoverColumn).setFormula(`=IMAGE("${albumCover}")`);
		} else {
			sheet.getRange(i + 2, albumCoverColumn).setValue("No cover available");
		}
	}
	
	// Re-enforce a fixed row height for all rows after processing
	for (let row = 2; row <= lastRow; row++) {
		sheet.setRowHeight(row, 200);
		sheet.getRange(row, discogsNotesColumn).setWrap(false);
	}

	// Throttle so we don't exceed ~60 requests/min 
	Utilities.sleep(1500); // Sleep 1.5 seconds after each iteration
	  
	} catch (error) {
		Logger.log(`Error fetching data for URL: ${url}`);
		Logger.log(error.toString());
	}
	}
}
```


The code is pretty self-explenatory with the comments and the variable naming I believe. Let me know if there's anything weird with it. I think the only limitation is our search speed and the Discogs API, which provides this piece of information on Rate Limiting our Discogs API App:
>  **Rate Limiting**
> 
> **Requests are throttled by the server by source IP to 60 per minute for authenticated requests, and 25 per minute for unauthenticated requests, with some exceptions.**
> 
> Your application should identify itself to our servers via a unique user agent string in order to achieve the maximum number of requests per minute.
> Our rate limiting tracks your requests using a moving average over a 60 second window. If no requests are made in 60 seconds, your window will reset.
> We attach the following headers to responses to help you track your rate limit use:
> >`X-Discogs-Ratelimit`: The total number of requests you can make in a one minute window.
> 
> > `X-Discogs-Ratelimit-Used` : The number of requests you’ve made in your existing rate limit window.
> 
> >`X-Discogs-Ratelimit-Remaining`: The number of remaining requests you are able to make in the existing rate limit window.

This is why we will add a bottleneck to our speed and limit the rate to 1 record every aprox. 1.5 seconds. 
```javascript
Utilities.sleep(1500); // Sleep 1.5 seconds after each iteration
```
