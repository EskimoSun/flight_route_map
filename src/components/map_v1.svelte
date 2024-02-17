<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import * as topojson from 'topojson-client';

  // Define width and height
  const width = 1000;
  const height = 600;

  let circles, lines; // Define variables to hold circle and line selections
  let airportData; // Define variable to hold airport data
  let suggestions = []; // Define variable to hold suggested airport codes
  let clickedCircles = new Set(); // Keep track of clicked circles
  let svg; // Define variable to hold the SVG element

  // Load and draw map data on component mount
  onMount(() => {
    // Define projection
    const projection = d3.geoMercator()
      .scale(140)
      .translate([width / 2, height / 1.4]);

    // Define path generator
    const path = d3.geoPath(projection);

    // Define zoom behavior
    const zoom = d3.zoom()
      .scaleExtent([1, 8]) // Define zoom extent
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

            // Draw routes
            lines = filteredRoutesData.map(route => {
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
              .attr('stroke', '#777777') // Light gray
              .attr('stroke-width', 0.2) // Solid line
              .attr('stroke-dasharray', '2 2') // Dotted line
              .attr('stroke-opacity', 0.2); // Half-transparent
              
            // Place circles above lines
            circles = svg.selectAll('circle')
              .data(airportData.filter(d => {
                // Filter only the airports mentioned in filteredRoutesData
                return lines.some(route => route.source_iata === d.iata || route.destination_iata === d.iata);
              }))
              .enter().append('circle')
              .attr('cx', d => projection([+d.long, +d.lat])[0])
              .attr('cy', d => projection([+d.long, +d.lat])[1])
              .attr('r', 1) // Increased size
              .attr('fill', d => clickedCircles.has(d.iata) ? 'blue' : '#444') // Initial color based on clicked status
              .on('mouseover', handleMouseOver)
              .on('mouseout', handleMouseOut)
              .on('click', handleMouseClick);

            // Add airport names as titles
            circles.append('title')
              .text(d => d.airport);
            
            // Place circles above lines
            circles.raise();
          });
        });
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
  }

  // Function to handle mouseover event
  function handleMouseOver(event, d) {
    const circle = d3.select(this);
    const iata = d.iata;

    // Highlight circle and its connected segment if clicked
    if (!clickedCircles.has(iata)) {
      circle.attr('fill', 'green').attr('r', 2).attr('opacity', 0.8); // Change circle color
      svg.selectAll('line').each(function(lineData) {
        const sourceIata = lineData.source_iata;
        const destinationIata = lineData.destination_iata;
        if ((sourceIata === iata || destinationIata === iata) && !clickedCircles.has(iata)) {
          d3.select(this)
            .attr('stroke', 'lime') // Change connected line color
            .attr('stroke-width', 0.8)
            .attr('stroke-opacity', 0.3)
            .attr('stroke-dasharray', null); // Change from dotted line to solid line
          // Find the other endpoint of the line and change its color
          const otherIata = sourceIata === iata ? destinationIata : sourceIata;
          circles.filter(c => c.iata === otherIata).attr('fill', 'red');
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
      circle.attr('fill', '#444').attr('r', 1); // Revert circle color
      svg.selectAll('line').each(function(lineData) {
        const sourceIata = lineData.source_iata;
        const destinationIata = lineData.destination_iata;
        if (sourceIata === iata || destinationIata === iata) {
          d3.select(this)
            .attr('stroke', '#777777') // Revert connected line color
            .attr('stroke-width', 0.2)
            .attr('stroke-opacity', 0.2)
            .attr('stroke-dasharray', '2 2'); // Revert back to dotted line
          // Find the other endpoint of the line and revert its color
          const otherIata = sourceIata === iata ? destinationIata : sourceIata;
          circles.filter(c => c.iata === otherIata).attr('fill', d => clickedCircles.has(otherIata) ? 'blue' : '#444');
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
      circle.attr('fill', 'blue').attr('r', 2); // Change circle color
      svg.selectAll('line').each(function(lineData) {
        const sourceIata = lineData.source_iata;
        const destinationIata = lineData.destination_iata;
        if (sourceIata === iata || destinationIata === iata) {
          d3.select(this)
            .attr('stroke', '#0055dd') // Change connected line color
            .attr('stroke-width', 0.8)
            .attr('stroke-opacity', 0.3)
            .attr('stroke-dasharray', null); // Change from dotted line to solid line
          // Find the other endpoint of the line and change its color
          const otherIata = sourceIata === iata ? destinationIata : sourceIata;
          circles.filter(c => c.iata === otherIata).attr('fill', '#00aaff');
        }
      });
    } else {
      clickedCircles.delete(iata);
      circle.attr('fill', '#444').attr('r', 1); // Revert circle color
      svg.selectAll('line').each(function(lineData) {
        const sourceIata = lineData.source_iata;
        const destinationIata = lineData.destination_iata;
        if (sourceIata === iata || destinationIata === iata) {
          d3.select(this)
            .attr('stroke', '#777777') // Revert connected line color
            .attr('stroke-width', 0.2)
            .attr('stroke-opacity', 0.2)
            .attr('stroke-dasharray', '2 2'); // Revert back to dotted line
          // Find the other endpoint of the line and revert its color
          const otherIata = sourceIata === iata ? destinationIata : sourceIata;
          circles.filter(c => c.iata === otherIata).attr('fill', d => clickedCircles.has(otherIata) ? 'blue' : '#444');
        }
      });
    }
  }

  // Function to handle search
  function handleSearch() {
    const searchTerm = document.getElementById('searchInput').value.trim().toUpperCase(); // Get search term
    if (searchTerm === '') {
      hideSuggestions();
      // If search term is empty, revert colors to default
      circles.attr('fill', '#444').attr('r', 1).attr('opacity', 1);
      svg.selectAll('line').attr('stroke', '#777777').attr('stroke-dasharray', '2 2').attr('stroke-opacity', 0.2).attr('stroke-width', 0.2);
      // Clear latitude and longitude
      document.getElementById('latLong').innerText = '';
      // Clear connected airports
      document.getElementById('connectedAirports').innerText = '';
      // Hide circle info
      d3.select('#tooltip')
        .style('opacity', 1);
    } else {
      // Filter airport codes that start with the search term
      suggestions = airportData.filter(d => d.iata.startsWith(searchTerm));
      // Show suggestions in the dropdown list
      showSuggestions();
      // If there's only one unique suggestion, hide suggestions and trigger search
      if (searchTerm.length === 2){
        circles.attr('fill', '#444').attr('r', 1).attr('opacity', 1);
        svg.selectAll('line').attr('stroke', '#777777').attr('stroke-dasharray', '2 2').attr('stroke-opacity', 0.2).attr('stroke-width', 0.2);
        // Clear latitude and longitude
        document.getElementById('latLong').innerText = '';
        // Clear connected airports
        document.getElementById('connectedAirports').innerText = '';
        // Hide circle info
        d3.select('#tooltip')
          .style('opacity', 1);  
      }    
      if (suggestions.length === 1) {
        document.getElementById('searchInput').value = suggestions[0].iata;
        hideSuggestions();
        executeSearch(suggestions[0].iata);
      }
    }
  }

  // Function to execute search
  function executeSearch(searchTerm) {
    // Highlight corresponding airport and segment
    circles.attr('opacity', 0.3).attr('fill', d => d.iata === searchTerm ? 'red' : '#444').attr('r', d => d.iata === searchTerm ? 2 : 1).attr('opacity', d => d.iata === searchTerm ? 1 : 0.3);
    svg.selectAll('line').each(function(lineData) {
      const sourceIata = lineData.source_iata;
      const destinationIata = lineData.destination_iata;
      if (sourceIata === searchTerm || destinationIata === searchTerm) {
        d3.select(this)
          .attr('stroke-width', 1)
          .attr('stroke-opacity', 1)
          .attr('stroke', 'red') // Change connected line color
          .attr('stroke-dasharray', null); // Change from dotted line to solid line
        const otherIata = sourceIata === searchTerm ? destinationIata : sourceIata;
        circles.filter(c => c.iata === otherIata).attr('fill', 'red');
      }
    });
    // Display latitude and longitude
    const airport = airportData.find(d => d.iata === searchTerm);
    if (airport) {
      document.getElementById('latLong').innerText = searchTerm + `: ${airport.airport} \n` + `Latitude: ${airport.lat}\n Longitude: ${airport.long}`;
    } else {
      document.getElementById('latLong').innerText = '';
    }
    // Show connected airports
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
      if (destinationAirport) {
        return {
          iata: airportIata,
          distance: getDistanceFromLatLonInKm(airport.lat, airport.long, destinationAirport.lat, destinationAirport.long)
        };
      }
    });

    // Remove undefined distances
    const validDistances = distances.filter(distance => distance);

    // Sort connected airports based on distance (from least to largest)
    validDistances.sort((a, b) => a.distance - b.distance);

    // Remove duplicate destinations
    const uniqueDestinations = [];
    validDistances.forEach(({ iata, distance }) => {
      if (!uniqueDestinations.some(dest => dest.iata === iata)) {
        uniqueDestinations.push({ iata, distance });
      }
    });

    // Display connected airports with distances
    const connectedAirportsList = document.getElementById('connectedAirports');
    connectedAirportsList.innerHTML = ''; // Clear previous list items
    uniqueDestinations.forEach(({ iata, distance }) => {
      const destinationAirport = airportData.find(airport => airport.iata === iata);
      if (destinationAirport) {
        const listItem = document.createElement('li');
        listItem.textContent = `${iata}: ${destinationAirport.airport} (${distance.toFixed(2)} km)`;
        connectedAirportsList.appendChild(listItem);
      }
    });

    // Show circle info for the searched airport
    const searchedAirport = airportData.find(d => d.iata === searchTerm);
    if (searchedAirport) {
      d3.select('#tooltip')
        .style('left', (searchedAirport.x + 10) + 'px')
        .style('top', (searchedAirport.y - 28) + 'px')
        .style('opacity', 1)
        .text(searchedAirport.airport);
    } else {
      d3.select('#tooltip')
        .style('opacity', 0);
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
    margin-top: 70px; /* Increased margin top */
    list-style: none;
    padding: 0;
    max-height: 200px; /* Set maximum height */
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
</style>

<!-- Search bar -->
<div id="tooltip"></div>
<div style="position: absolute; top: 20px; right: 20px;">
  <input type="text" id="searchInput" on:input={handleSearch} placeholder="Search airport...">
  <div id="searchLabel"></div> <!-- Suggestions will be displayed here -->
</div>

<!-- Latitude and Longitude -->
<div id="latLong" style="position: absolute; top: 50px; right: 20px; background-color: white;"></div>

<!-- Connected airports -->
<ul id="connectedAirports" style="position: absolute; top: 70px; right: 20px;"></ul>