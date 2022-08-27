---
title: "Google Summer of code'21"
layout: post
date: 2021-08-21
image: /assets/images/GSoC.png
headerImage: true
tag:
- opensource
- react
- typescript
- jsonschema
- gsoc
category: blog
author: sh15h4nk
description: Google summer of code 21, Freifunk
---
A peak into my google summer of code project
<!--more-->



## Before we begin

Hey everyone, I'm happy that my proposal has been accepted for GSoC'21 @freifunk. It's a bit sad that time flew very quick, in the past 10 week I worked on a project, which is intended to update the spec schema file from draft 3 version to latest jsonschema version, update the generator tool to work with latest version and update all the remanining tools which are dependent on spec schema files.

### Discrepancy
<span class="evidence">The current latest version of JSON schema is version [2012-12](https://json-schema.org/specification.html), but unfortunately there isn't much [support](https://json-schema.org/implementations.html#web-ui-generation) to update the dependent tools(to be specific, the [generator](https://freifunk.net/api-generator/)). So my mentor [Andibraeu](https://github.com/andibraeu) and I have chosen to work with draft7 version, as it has upper hand with implementation support compared to other recent versions.</span>


## Our initial layout
- Migrate spec schema files
- Picking generator
- Generate and test forms
- Update all the dependent tools
- Test updated tools
- Fix bugs, if any

## Phase I summary
In the first phase of the period I mostly worked on the generator, And I have only migrated the recent version of spec file, so that we can immediately start working with the tools. In the beginning no framework or implementation seemed perfect for the generator so, I have picked some of the frameworks to try and test, and in the end we weighed pros and cons to finally choose one.


### Spec file
Migrating the spec files to draft 7 version is not a difficult task. I have migrated the 0.4.16.json file to draft 7, and stated working on the generator and all the other tools, And after updating all the tools, I have migrated all the spec files to draft 7 version.

**Some features to point out:**
- Used [definitions](https://json-schema.org/understanding-json-schema/structuring.html#defs) to reuse some fields like (phone, email, city, URL, etc.).
- Added [formats](https://json-schema.org/understanding-json-schema/reference/string.html#built-in-formats) to some fields like (url, email, lastChange).
- Used [minProperties](https://json-schema.org/understanding-json-schema/reference/object.html#size) instead of required for contacts.
- Used [AdditionalProperties](https://json-schema.org/understanding-json-schema/reference/object.html#additional-properties) to block extra fields.


### Generator 
This is one of the significant tool depending on spec schema files. The job of the tool is very simple, it takes JSON schema as input, generates HTML forms to render in the browsers, handles validation of the form data against the input JSON schema and finally, generates a JSON out file.

<img class="image" src="/assets/images/generator.png" />



### Frameworks Evaluation
<span class="pick-generator">I have choosen the following frameworks to build and test the form</span>
1. UI Schema for React
2. React JSON schema forms(mozilla)
3. Restspace schema forms
4. JSON Forms(jsonforms.io, Eclipse source) 

I choose to work with React beacuse all these frameworks are proficient with the react and vue.
All the code for the building and testing of the frameworks can be found [here](https://github.com/sh15h4nk/json-webui-generator)
At last, we have choose jsonforms by considering all the pros and cons.
A detailed document of pros and cons can be found [here](https://docs.google.com/document/d/1dhPFyQ65YD8_dSjbzyoyvSfXyZUhtRU4DL17XdTOIi4/edit?usp=sharing).

---

There after I started to implement the features of the Generator tool like, showing the form data(bound data), Validation of the form data, loading data to the form.



#### Bound data
As I mentioned I was using React to build the forms with the frameworks, so in react a niche method to handle data is via **hooks**. So this was a easy task, I rendered the grid containing form data, beside to the form. And the grid is updated on update of the state.
<img class="image" src="/assets/images/bound-data.png"/> 




#### Validation and Submission
Here, the JSONForms only emmit errors via an event, so on the event emmision I had to update the state, event I had to do the same thing to update the form data, it emmits OnChange event when the form data is updated along with the errors. So I have added a function to record errors and show the validation errors.

```tsx
<JsonForms
    schema={schema}
    uischema={uischema}
    data={jsonformsData}
    renderers={renderers}
    cells={materialCells}
    validationMode={"ValidateAndShow"}
    onChange={({ errors, data }) => {
        setJsonformsData(data);
        recordErrors(errors);
        if (Object.keys(jsonformsData).length === 0) recordErrors([]);
       }
    }
/>
```

And for the submission of form I had to consider that there are no recorded errors and the form data should not be empty because I have recorded errors into state only if the forms data is not empty.


Errors are emitted by the event even before starting to fill the form. By validating all this checks, I have generated the output JSON file.

<img class="image" src="/assets/images/validate-data.png"/> 




#### Loading Data
This was also easy to achieve, I just had to write a function which fetches the API file of the comminity and set the state. But most of the online webservers doesn't allow cors. So I cannot fetch the files through the browser, as it blocks cors, so I had to use a proxy server to fetch the data for me and load that data into the form. This is also the same method used in the pervious version of the generator. So I used the old php proxy server.

```tsx
let loadData = (community: any) => {
// console.log(community.value)
fetch("https://freifunk.net/api/generator/php-simple-proxy/ba-simple-proxy.php?url="+comminutiesFiles[community.value])
	.then(response => response.json())
	.then(data => {
	data = correctLocation(data.contents);
	data = correctCaseSensitivity(data);
	setJsonformsData(data);
	})
	.catch(()=> console.log("NetworkError: Can't fetch the api file"))
}
```

But before that I had to fetch all the communities and their url links, and list all the communities in a select filed. So I have used directory.json to fetch all the communities.

```tsx
 //fetching all the API files
const [comminutiesFiles, setCommunitiesFiles ] = useState([]);
const fetchCommunities = () => {
fetch("https://raw.githubusercontent.com/freifunk/directory.api.freifunk.net/master/directory.json")
	.then(response => response.json())
	.then(data => setCommunitiesFiles(data))
	.catch(()=> console.log("NetworkError: Can't fetch the api file"))
}
useEffect( () => {
fetchCommunities();
},[])

//list of communities to select
const communities: Array<{value: string, label: string}> = []
Object.keys(comminutiesFiles).sort().forEach((comm)=> {
communities.push({ value: comm, label: comm})
})
```


Then I have used React Select to render a select field.\
<img class="image" src="/assets/images/load-data.png"/> 

*At the end of the phase I, I have completed finished choosing the framework for generator and build form with the draft 7 spec schema file*



## Phase II summary
In the final phase of period, I have worked with the location picking feature of the generator tool and updating the dependent tool. I had to develop a custom renderer for the fields lat, lon in the location and additionalLocations properties inorder to render a map instead of text fields, but for that I had to embed the lat, lon fields into an object, that change in the spec file lead us to add a new version of the spec file. And the update of the lat, lon in the spec file effects some other tools like [Community finden](https://freifunk.net/wie-mache-ich-mit/community-finden/), [Kontakt](https://freifunk.net/kontakt/), etc, so we have to update them too, but luckily all those are tools use summarized API file, And the [collector script](https://github.com/freifunk/common.api.freifunk.net/blob/master/collector/collectCommunities.py) is used to generate the summarized file.



### Custom Renderer
In order to render lat, lon fields as a map to pick them up. I have written a custom renderer which particularly consists of three parts
- **A tester**\
	This is something that tells to JSONForms to use a custom control to render the specifed field.
	```tsx
	import { rankWith, scopeEndsWith } from '@jsonforms/core';

	export default rankWith(
		100,
		scopeEndsWith('geoCode')//location picker
	)
	```

- **A control**\
	The control is used to incorporate the form data and the data of the renderer(component) with withJsonFormsControlProps.

	```tsx
	import * as React from 'react';
	import { withJsonFormsControlProps } from "@jsonforms/react";
	import { MapContainer, TileLayer } from 'react-leaflet';
	import { Location }  from './Location';
	import { useState } from 'react';
	import 'leaflet/dist/leaflet.css';

	interface LocationControlProps {
		data: any;
		handleChange(path: string, value: any): void;
		path: string;
	}

	let center = {
	lat: 51.1657,
	lng: 10.4515
	}

	const LocationControl: React.FC<LocationControlProps> = ({ data, handleChange, path }) => {
	const [mapCenter, setMapcenter] = useState(center);

	if(data && !(mapCenter.lat === data.lat && mapCenter.lng === data.lon)){
		const t = {lat: data.lat, lng: data.lon }
		setMapcenter(t);
	}

	return(
		<MapContainer center={mapCenter} zoom={7} scrollWheelZoom={false} style={{ height: '50vh', width: '100wh' }}>
		<TileLayer
			attribution='&copy; <a href="http://osm.org/copyright">OpenStreetMap</a> contributors'
			url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
		/>
		<Location 
			value = {data}
			updateCords={(newValue: Object) => handleChange(path, newValue)}
		/>
		</MapContainer>
	)
	};

	export default withJsonFormsControlProps(LocationControl);
	```

- **A renderer (React component)**\
	This is just a react component returns the map to render on the page.
	The major issue that I have faced here was, I couldn't load the data to the renderer accordingly, as this certainly has no link with the jsonformdata state. But eventually I figured a way, have look at the above code the way I used to set the position, I feel it was a but tricky.
	
	```tsx
	import * as React from 'react';
	import { Marker } from 'react-leaflet';
	import { useState, useRef, useMemo } from 'react';

	import 'leaflet/dist/leaflet.css';
	import 'leaflet/dist/leaflet.js';

	import L from 'leaflet';


	L.Icon.Default.imagePath='leaflet_images/';

	interface LocationProps {
	id?: string;
	value: {lat: number, lon: number};
	updateCords: (newValue: Object) => void;
	}

	let center = {
	lat: 51,
	lng: 8
	}

	export const Location: React.FC<LocationProps> = ({ id, value, updateCords }) => {
	const [position, setPosition] = useState(center);
	const markerRef = useRef(null)
	const eventHandlers = useMemo(
		() => ({
		dragend() {
			const marker: any = markerRef.current;
			if (marker != null) {
			var cords = marker.getLatLng();
			setPosition(cords);
			// console.log("The marker", cords);
			updateCords({lat: cords.lat, lon: cords.lng});
			}
		},
		}),
		[updateCords],
	)
	if (value && !(value.lat === position.lat && value.lon === position.lng)){
		const t = {lat: value.lat, lng: value.lon};
		setPosition(t);
	}

	return (
		<Marker
		draggable={true}
		eventHandlers={eventHandlers}
		position={position}
		ref={markerRef}>
		</Marker>
	)
	}
	```


### Dependent tools
The tools that are purely dependent of the spec schema files are API viewer and Travis Job on the directory repository. But upon disturbing the lat, lon fields in the schema files, I have raised issues with the other tools. But those tools depend on a summarized API file, which is a collection of the all the communities API files, so I have to update the script which generates the summarized API file.

**Tools**
- **API Viewer**\
	This tool generates a static build of pages, which show the validation result of the communities API file data.
	<img class="image" src="/assets/images/api-viewer1.png" />\\
	Valid API file
	<img class="image" src="/assets/images/api-viewer2.png" />\\
	Invalid API file
	<img class="image" src="/assets/images/api-viewer3.png" />\\
	Validation Errors
	<img class="image" src="/assets/images/api-viewer4.png" />\\
	- ***Improvements:***
		- The prior tool existed in python 2, I have updated to python 3.
		- Update to Validate data against draft 7 schema and show validation.
		- Added [datatables](https://datatables.net/) to list the communities.


- **Travis Job**\
	All the API files are collected in the [directory repository](https://github.com/freifunk/directory.api.freifunk.net). And this Travis job validates the data of the API file data, when they are updated or added to the [directory.json](https://github.com/freifunk/directory.api.freifunk.net/blob/master/directory.json)

	<img class="image" src="/assets/images/travis-job(1).png" />\\
	Travis Job console output
	<img class="image" src="/assets/images/travis-job(2).png" />\\
	Job result
	<img class="image" src="/assets/images/travis-job(3).png" />\\
	A build of the travis job can be found [here](https://app.travis-ci.com/github/freifunk/directory.api.freifunk.net/builds/235043001).
	- ***Improvements:***
		- Also, this tool (test) existed in python 2, I have updated it to python 3.
		- Updated to validate data against draft 7.


- **Common API (collector script)**\
	If you recall as I have added new spec file by embedding latitude, longitude into an object to adopt with the jsonforms custom renderer for map picker using react-leaflet. This would affect a lot of other tools like [Community finden](https://freifunk.net/wie-mache-ich-mit/community-finden/), [Kontakt](https://freifunk.net/kontakt/), etc which are truly based on the lon, lat fields of the API files. But luckily all these tools use a summarized API file, And the [collector script](https://github.com/freifunk/common.api.freifunk.net/blob/master/collector/collectCommunities.py) is used to collect all the communities files.

	- ***Improvements:***
		- Deserialized geoCode object and appended the fields to the respective locations. So that the fields are set to their old locations.

## Evaluations 
The Organisation wanted us to write blog posts at different stages of the timeline
- [Before coding period](https://blog.freifunk.net/2021/06/13/gsoc21-json-schema-webui-generator/)
- [First Evaluation](https://blog.freifunk.net/2021/07/11/gsoc21-json-schema-webui-generator-ii/)
- [Final Evaluation](https://blog.freifunk.net/2021/08/21/gsoc21-api-generator-and-tools-with-draft-7-json-schema/)

## References
- **Spec Schema Files**
	- [Draft 7 Migration](https://github.com/freifunk/api.freifunk.net/pull/164)
	- [Patch](https://github.com/freifunk/api.freifunk.net/pull/165)
	- [Patch](https://github.com/freifunk/api.freifunk.net/pull/166)
	<!-- - [Patch]() -->

- **Generator**
	- [Framework Implementations](https://github.com/sh15h4nk/json-webui-generator)
	- [Framework Evaluation](https://docs.google.com/document/d/1dhPFyQ65YD8_dSjbzyoyvSfXyZUhtRU4DL17XdTOIi4/edit?usp=sharing)
	- [API Generator](https://github.com/freifunk/generator.api.freifunk.net)
	- [live](https://freifunk.github.io/generator.api.freifunk.net/)

- **API Viewer**
	- [Python 3 migration and draft 7 validation](https://github.com/freifunk/viewer.api.freifunk.net/pull/4)
	- [Patch](https://github.com/freifunk/viewer.api.freifunk.net/pull/5)

- **Travis Job**
	- [Python 3 migration and draft 7 validation](https://github.com/freifunk/directory.api.freifunk.net/pull/673)

- **Common API**
	- [Altering location fields](https://github.com/freifunk/common.api.freifunk.net/pull/12)

## Conclusion
I have started the project with minimal understanding of react, typescript and jsonschema. But it was very fun to understand and work on. I really liked this way to learn new things rather than reading or doing a course. Every issue that I have encountered had leaded me to understand the things briefly. I'm really thankful to freifunk for the opportunity. And a big shout out to my mentor Andreas Bräu for absolutely wonderful guidance and support.

