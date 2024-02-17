<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import * as topojson from 'topojson-client';

  // Define width and height
  const width = 1100;
  const height = 600;

  let circles, lines; // Define variables to hold circle and line selections
  let airportData; // Define variable to hold airport data
  let suggestions = []; // Define variable to hold suggested airport codes
  let clickedCircles = new Set(); // Keep track of clicked circles
  let svg; // Define variable to hold the SVG element
  let selected_x; // Define variable to record selected circle x position
  let selected_y; // Define variable to record selected circle y position

  // Load and draw map data on component mount
  onMount(() => {
    // Define projection
    const projection = d3.geoMercator()
      .scale(170)
      .translate([width / 2, height / 1.4]);

    // Define path generator
    const path = d3.geoPath(projection);

    // Define zoom behavior
    const zoom = d3.zoom()
      .scaleExtent([1, 4]) // Define zoom extent
      .on('zoom', handleZoom);

    // Load world map data
    d3.json('https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json')
      .then(worldData => {
        // Draw world map
        const countries = topojson.feature(worldData, worldData.objects.countries);
        svg = d3.select('svg')
          .attr('width', width)
          .attr('height', height)
          .call(zoom) // Call zoom behavior on SVG element
          .append('g');

        svg.selectAll('path')
          .data(countries.features)
          .enter().append('path')
          .attr('class', 'country')
          .attr('d', path);

        // Load airport data
        d3.csv('/src/components/data/airports_info_final.csv').then(data => {
          airportData = data; // Store airport data in the global variable
          
          // Load routes data
          d3.csv('/src/components/data/route_final.csv').then(routesData => {
            // Filter routes data based on 'number_airport' column
            const filteredRoutesData = routesData.filter(route => route.number_airport > 150);

            // Create data for all filtered routes
            lines = filteredRoutesData.map(route => {
              // only use routes with airport data in airportData
              const sourceAirport = airportData.find(airport => airport.iata === route.source_airport);
              const destinationAirport = airportData.find(airport => airport.iata === route.destination_airport);
              if (sourceAirport && destinationAirport) {
                const sourceCoords = projection([+sourceAirport.long, +sourceAirport.lat]);
                const destCoords = projection([+destinationAirport.long, +destinationAirport.lat]);
                return {
                  source_iata: route.source_airport,
                  destination_iata: route.destination_airport,
                  sourceCoords,
                  destCoords
                };
              }
            }).filter(route => route); // Filter out undefined routes

            // Draw lines
            svg.selectAll('line')
              .data(lines)
              .enter().append('line')
              .attr('x1', d => d.sourceCoords[0])
              .attr('y1', d => d.sourceCoords[1])
              .attr('x2', d => d.destCoords[0])
              .attr('y2', d => d.destCoords[1])
              .classed('init', true); // Initial attributes
              
            // Draw circles
            circles = svg.selectAll('circle')
              .data(airportData.filter(d => {
                // Filter only the airports mentioned in filteredRoutesData
                return lines.some(route => route.source_iata === d.iata || route.destination_iata === d.iata);
              }))
              .enter().append('circle')
              .attr('cx', d => projection([+d.long, +d.lat])[0])
              .attr('cy', d => projection([+d.long, +d.lat])[1])
              .classed('init', true) // Initial attributes
              .on('mouseover', handleMouseOver)
              .on('mouseout', handleMouseOut)
              .on('click', handleMouseClick);

            // Add airport names as titles
            circles.append('title')
              .text(d => d.airport);
          });
        });

        d3.select('#tooltip').style('opacity', 0);
      })
      .catch(error => {
        console.error('Error loading map data:', error);
      });
  });

  // Function to handle zoom behavior
  function handleZoom(event) {
    const { transform } = event;
    svg.selectAll('path').attr('transform', transform); // Apply zoom transform to paths
    svg.selectAll('circle').attr('transform', transform); // Apply zoom transform to circles
    svg.selectAll('line').attr('transform', transform); // Apply zoom transform to lines

    // Tramsform tooltip position
    const transform_tooltip = d3.zoomTransform(svg.node());
    d3.select('#tooltip')
      .style("left", (transform_tooltip.applyX(selected_x) + 20 + "px"))
      .style("top", (transform_tooltip.applyY(selected_y) + 30 + "px"));
  }

  // Function to handle mouseover event
  function handleMouseOver(event, d) {
    const circle = d3.select(this);
    const iata = d.iata;

    // Highlight circle and its connected segment if clicked
    if (!clickedCircles.has(iata)) {
      circle.classed('hover', true); // Hover attributes

      svg.selectAll('line').each(function(lineData) {
        const sourceIata = lineData.source_iata;
        const destinationIata = lineData.destination_iata;
        if ((sourceIata === iata || destinationIata === iata) && !clickedCircles.has(iata)) {
          d3.select(this).classed('hover', true); // Hover attributes

          // Find the other endpoint of the line and change its color
          const otherIata = sourceIata === iata ? destinationIata : sourceIata;
          circles.filter(c => c.iata === otherIata).classed('connected', true); // Connected attributes
        }
      });
    }
    // Show tooltip with airport name
    d3.select('#tooltip')
      .style('left', (event.pageX + 10) + 'px')
      .style('top', (event.pageY - 28) + 'px')
      .style('opacity', 1)
      .text(d.airport);
  }

  // Function to handle mouseout event
  function handleMouseOut(event, d) {
    const circle = d3.select(this);
    const iata = d.iata;

    // Revert circle and segment color if not clicked
    if (!clickedCircles.has(iata)) {
      circle.classed('hover', false); // Remove hover attributes

      svg.selectAll('line').each(function(lineData) {
        const sourceIata = lineData.source_iata;
        const destinationIata = lineData.destination_iata;
        if (sourceIata === iata || destinationIata === iata) {
          d3.select(this).classed('hover', false); // Remove hover attributes

          // Find the other endpoint of the line and revert its color
          const otherIata = sourceIata === iata ? destinationIata : sourceIata;
          circles.filter(c => c.iata === otherIata).classed('connected', false); // Remove connected attributes
        }
      });
    }

    // Hide tooltip
    d3.select('#tooltip')
      .style('opacity', 0);
  }

  // Function to handle click event
  function handleMouseClick(event, d) {
    const circle = d3.select(this);
    const iata = d.iata;

    // Toggle click status and update color
    if (!clickedCircles.has(iata)) {
      clickedCircles.add(iata);
      circle.classed('active', true); // Active attributes

      svg.selectAll('line').each(function(lineData) {
        const sourceIata = lineData.source_iata;
        const destinationIata = lineData.destination_iata;
        if (sourceIata === iata || destinationIata === iata) {
          d3.select(this).classed('active', true); // Change from dotted line to solid line

          // Find the other endpoint of the line and change its color
          const otherIata = sourceIata === iata ? destinationIata : sourceIata;
          circles.filter(c => c.iata === otherIata).classed('connected-active', true); // Connected-active attributes
        }
      });

      // Show information of airport
      executeSearch(iata);
    } else {
      clickedCircles.delete(iata);
      circle.classed('active', false); // Remove active attributes

      svg.selectAll('line').each(function(lineData) {
        const sourceIata = lineData.source_iata;
        const destinationIata = lineData.destination_iata;
        if (sourceIata === iata || destinationIata === iata) {
          d3.select(this).classed('active', false); // Revert back to dotted line

          // Find the other endpoint of the line and revert its color
          const otherIata = sourceIata === iata ? destinationIata : sourceIata;
          circles.filter(c => c.iata === otherIata).classed('connected-active', false); // Remove connected-active attributes
        }
      });

      // Clear information elements
      document.getElementById('latLong').innerText = '';
      document.getElementById('connectedAirports').innerText = '';
    }
  }

  // Function to handle search
   function handleSearch(event) {
    // Get inputted search term
    const searchTerm = document.getElementById('searchInput').value.trim().toUpperCase(); 
    // Filter airport codes that start with the search term
    suggestions = airportData.filter(d => d.iata.startsWith(searchTerm));
    
    // Hide suggestions on default, show suggestions when inputted
    if (searchTerm === ''){
      hideSuggestions();
    }else{
      showSuggestions();
    }

    // When inputted 3 letters, hide suggestions and trigger search, otherwise reset everything
    if (searchTerm.length < 3) {
      // Revert circles and lines to initial attributes
      circles.classed('active', false).classed('connected-active', false);
      svg.selectAll('line').classed('active', false);
      
      // Clear elements
      document.getElementById('latLong').innerText = '';
      document.getElementById('connectedAirports').innerText = '';
      d3.select('#tooltip')
        .style('opacity', 0);
    } else {
      // Detect 'Enter' key press
      if (event.key === 'Enter') {
        document.getElementById('searchInput').value = suggestions[0].iata;
        hideSuggestions();
        executeSearch(suggestions[0].iata);
      }
    }
  }

  // Function to execute search
  function executeSearch(searchTerm) {
    // Highlights searched airport and connected routes
    circles.filter(d => d.iata === searchTerm).classed('active', true); // Active attributes
    
    svg.selectAll('line').each(function(lineData) {
      const sourceIata = lineData.source_iata;
      const destinationIata = lineData.destination_iata;
      if (sourceIata === searchTerm || destinationIata === searchTerm) {
        d3.select(this).classed('active', true); // Active attributes

        const otherIata = sourceIata === searchTerm ? destinationIata : sourceIata;
        circles.filter(c => c.iata === otherIata).classed('connected-active', true); // Connected-active attributes
      }
    });

    // Display latitude and longitude
    const airport = airportData.find(d => d.iata === searchTerm);
    if (airport) {
      document.getElementById('latLong').innerText = searchTerm + `: ${airport.airport} \n` + `Latitude: ${airport.lat}\n Longitude: ${airport.long}`;
    } else {
      document.getElementById('latLong').innerText = '';
    }

    // Search connected airports
    const connectedAirports = lines.reduce((connected, line) => {
      const sourceIata = line.source_iata;
      const destinationIata = line.destination_iata;
      if (sourceIata === searchTerm) connected.push(destinationIata);
      else if (destinationIata === searchTerm) connected.push(sourceIata);
      return connected;
    }, []);

    // Calculate distances to connected airports
    const distances = connectedAirports.map(airportIata => {
      const destinationAirport = airportData.find(airport => airport.iata === airportIata);
      const dist = getDistanceFromLatLonInKm(airport.lat, airport.long, destinationAirport.lat, destinationAirport.long);
      if (destinationAirport) {
        return {
          airport: destinationAirport.airport,
          iata: airportIata,
          distance: dist,
        };
      }
    }).filter(distance => distance).sort((a, b) => a.distance - b.distance); // Remove undefined and sort

    // Display connected airports with distances
    const connectedAirportsList = document.getElementById('connectedAirports');
    connectedAirportsList.innerHTML = ''; // Clear previous list items
    distances.forEach(({airport, iata, distance }) => {
      // Create new element for each connected airport
      const listItem = document.createElement('li');
      listItem.textContent = `${iata}: ${airport} (${distance.toFixed(2)} km)`;
      connectedAirportsList.appendChild(listItem);
    });

    // Show circle info for the searched airport
    const searchedAirport = circles.filter(d => d.iata === searchTerm)._groups[0][0];
    if (searchedAirport) {
      d3.select('#tooltip')
        .style('left', (searchedAirport.cx.animVal.value + 20 + 'px'))
        .style('top', (searchedAirport.cy.animVal.value + 30 + 'px'))
        .style('opacity', 1)
        .text(airport.airport);
    } else {
      d3.select('#tooltip')
        .style('opacity', 0);
    };

    selected_x = searchedAirport.cx.animVal.value;
    selected_y = searchedAirport.cy.animVal.value;
  };

  function handleKeyPress(event) {
    if (event.key === 'Enter') {
      // If Enter key is pressed, execute the search
      const searchTerm = document.getElementById('searchInput').value.trim().toUpperCase();
      if (searchTerm.length === 3 && suggestions.length === 1) {
        // If the search term is 3 characters long and there's only one suggestion, execute the search
        document.getElementById('searchInput').value = suggestions[0].iata;
        hideSuggestions();
        executeSearch(suggestions[0].iata);
      }
    }
  }

  // Function to show suggestions
  function showSuggestions() {
    const searchLabel = document.getElementById('searchLabel');
    searchLabel.innerHTML = ''; // Clear previous suggestions
    suggestions.forEach(airport => {
      const suggestionItem = document.createElement('div');
      suggestionItem.textContent = airport.iata;
      suggestionItem.addEventListener('click', () => {
        // When a suggestion is clicked, fill the search input and trigger search
        document.getElementById('searchInput').value = airport.iata;
        hideSuggestions();
        executeSearch(airport.iata);
      });
      searchLabel.appendChild(suggestionItem);
    });
    // Show suggestions
    searchLabel.style.display = 'block';
  }

  // Function to hide suggestions
  function hideSuggestions() {
    // Hide suggestions
    document.getElementById('searchLabel').style.display = 'none';
  }

  // Function to calculate distance between two points on the globe
  function getDistanceFromLatLonInKm(lat1, lon1, lat2, lon2) {
    const R = 6371; // Radius of the earth in km
    const dLat = deg2rad(lat2 - lat1); // deg2rad below
    const dLon = deg2rad(lon2 - lon1);
    const a =
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.cos(deg2rad(lat1)) * Math.cos(deg2rad(lat2)) *
      Math.sin(dLon / 2) * Math.sin(dLon / 2);
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    const d = R * c; // Distance in km
    return d;
  }
  // Function to convert degrees to radians
  function deg2rad(deg) {
    return deg * (Math.PI / 180);
  }
</script>

<svg width={width} height={height}>
  <g></g>
</svg>

<style>
  /* Import styles from map.css */
  @import url('map.css');

  /* Additional styles specific to this component */
  #tooltip {
    position: absolute;
    background-color: rgba(255, 255, 255, 0.5); /* Half-transparent white background */
    border: 1px solid black;
    padding: 5px;
    border-radius: 5px;
    pointer-events: none;
  }

  /* Styling for search bar */
  #searchInput {
    padding: 10px;
    border-radius: 5px;
    border: 1px solid #ccc;
    font-size: 16px;
    outline: none;
    transition: border-color 0.3s ease-in-out;
    width: 300px; /* Adjust width as needed */
    background-color: white; /* Set background color to white */
  }

  #searchInput:focus {
    border-color: #007bff; /* Change border color on focus */
  }

  /* Style for latitude and longitude display */
  #latLong {
    font-size: 14px;
    margin-top: 30px; /* Increased margin top */
    width: 300px;
  }

  #connectedAirports {
    font-size: 14px;
    margin-top: 100px; /* Increased margin top */
    list-style: none;
    padding: 0;
    max-height: 400px; /* Set maximum height */
    overflow-y: auto; /* Enable vertical scrolling */
    border: 1px solid #ccc; /* Add a border for better visibility */
    border-radius: 5px; /* Add some border radius */
    width: 300px; /* Same width as search bar */
    background-color: white; /* Set background color to white */
  }

  /* Style for search suggestions */
  #searchLabel {
    display: none;
    position: absolute;
    background-color: white; /* White background */
    border: 1px solid #ccc;
    border-top: none;
    border-radius: 0 0 5px 5px;
    width: 300px; /* Same width as search bar */
    z-index: 1; /* Ensure suggestions appear above other elements */
    box-shadow: 0px 8px 16px 0px rgba(0,0,0,0.2); /* Add box shadow */
  }

  #writeup{
    font-size: 15px;
    margin-left: 20px;
    margin-right: 20px;
  }
</style>

<!-- Search bar -->
<div id="tooltip"></div>
<div style="position: absolute; top: 20px; right: 20px;">
  <input type="text" id="searchInput" on:input={(event) => handleSearch(event)} on:keydown={(event) => handleKeyPress(event)} placeholder="Search airport...">
  <div id="searchLabel"></div> <!-- Suggestions will be displayed here -->
</div>

<!-- Latitude and Longitude -->
<div id="latLong" style="position: absolute; top: 50px; right: 20px; background-color: white;"></div>

<!-- Connected airports -->
<ul id="connectedAirports" style="position: absolute; top: 70px; right: 20px;"></ul>

<div id="writeup">
  <span></span>
  <p>
    The primary aim of this visualization is to create an interactive global airport map using D3.js and Svelte. The map is designed to be visually appealing, user-friendly, and informative regarding the global airport network. All design decisions were made with the intention of optimizing the user experience and usability. The construction process involved two main components: data presentation and interaction techniques.
  </p>
  <p>
    In terms of data presentation, airport information and routes were carefully selected and processed for optimal web page performance. The Mercator projection was chosen for drawing the world map due to its ability to accurately represent angles, shapes, and distances between regions, aligning well with the flight route data, compared with other d3 projections like geoAzimuthalEqualArea and geoAlbers. The map utilizes a simple and clear visualization approach, using circles to denote airports and lines to illustrate routes between them. Initially, a gray color scheme was employed to maintain visual clarity before user interaction.
  </p>
  <p>
    Regarding interaction techniques, three main aspects were considered. Firstly, zoom functionality was incorporated to facilitate exploration of detailed map areas due to the abundance of data. Secondly, interaction with map data points, including hover and click actions, was improved through visual cues like color changes, enhancing user comprehension and engagement. Tooltips were also introduced to offer additional airport information upon hover, providing users with detailed insights without requiring additional clicks, thus enhancing the overall experience. Notably, selected airports remain highlighted, with their selection order displayed, facilitating the creation of flight paths with multiple stops. Lastly, a search bar was implemented to enable users to quickly locate specific airports and associated routes by IATA code, with autocomplete suggestions provided for ease of use and cognitive load reduction. Additionally, a formula calculating the distance between latitude and longitude was included to illustrate the spatial relationship between airports, adding further value to the map. Two options were considered for search bar interaction: automatically displaying search results upon finding an airport or showing results only upon user action (clicking or entering). To better meet user needs, the decision was made to adopt the second option.
  </p>
  <p>
    During the map development process, we collaborated to source and import data onto the web page, which was completed swiftly. The majority of that process time was dedicated to finding data and determining the visualization objectives, totaling approximately one and a half hours. Regarding interaction implementation, David was in charge of the interaction techniques for the data points on the map mentioned above, particularly focusing on hover and click interactions. This phase was the most time-consuming, spanning about four hours due to the complexity of the code and numerous opportunities for enhancement. Zhiqing primarily undertook tasks related to map zoom functionality and retrieval features, such as integrating the drop-down suggestion list, with a time investment of approximately three hours. Overall, the collaboration resulted in a comprehensive and user-friendly interactive airport map.
  </p>
</div>