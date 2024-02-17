<script>
  import { onMount, onDestroy } from 'svelte';
  import * as d3 from 'd3';
  import * as topojson from 'topojson-client';

  // Define width and height
  const width = 900;
  const height = 600;

  let circles, lines; // Define variables to hold circle and line selections
  let flattenedData;
  let suggestions = [];

  // Load and draw map data on component mount
  onMount(() => {
    // Define projection
    const projection = d3.geoMercator()
      .scale(140)
      .translate([width/2, height/1.4]);

    // Define path generator
    const path = d3.geoPath(projection);

    // Load world map data
    d3.json('https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json')
      .then(worldData => {
        // Draw world map
        const countries = topojson.feature(worldData, worldData.objects.countries);
        d3.select('svg g')
          .selectAll('path')
          .data(countries.features)
          .enter().append('path')
          .attr('class', 'country')
          .attr('d', path);

        // load combined airport and routes data
        d3.csv('/src/components/data/only_routes.csv').then(routesData => {
          // mark routes with lines
          lines = d3.select('svg g')
            .selectAll('line')
            .data(routesData)
            .enter().append('line')
            .attr('x1', d => projection([+d['Source long'], +d['Source lat']])[0])
            .attr('y1', d => projection([+d['Source long'], +d['Source lat']])[1])
            .attr('x2', d => projection([+d['Dest long'], +d['Dest lat']])[0])
            .attr('y2', d => projection([+d['Dest long'], +d['Dest lat']])[1])
            .classed('init', true); //set opacity to 40%
          
          // Step 1: Flatten routesData
          flattenedData = routesData.reduce((acc, d) => {
              // Create an entry for the source
              acc.push({
                  iata: d['Source iata'],
                  long: d['Source long'],
                  lat: d['Source lat'],
                  type: 'source', // Optional, helps identify the point type
                  ...d // Spread the rest of the data if needed
              });
              // Create an entry for the destination
              acc.push({
                  iata: d['Dest iata'],
                  long: d['Dest long'],
                  lat: d['Dest lat'],
                  type: 'destination', // Optional, helps identify the point type
                  ...d // Spread the rest of the data if needed
              });
              return acc;
          }, []);

          // Step 2: Bind Flattened Data to Circles
          const circles = d3.select('svg g')
              .selectAll('circle')
              .data(flattenedData)
              .enter().append('circle')
              .attr('cx', d => projection([+d.long, +d.lat])[0])
              .attr('cy', d => projection([+d.long, +d.lat])[1])
              .classed('init', true);
              //.attr('fill', d => d.type === 'source' ? '#ff6600' : '#007bff'); // Different color for source and destination, if desired
          
          // Add mouseover event listeners to circles
          circles.on('mouseover', function(event, d) { 
              // Hide all points 
              //circles.classed('hidden', true);

              const circle = d3.select(this);
              circle.classed('hover', true); // Change circle color
              
              d3.selectAll('line').each(function(lineData) {
                const sourceIata = lineData['Source iata'];
                const destinationIata = lineData['Dest iata'];
                if (destinationIata === d['iata'] || sourceIata === d['iata']) { // Find routes that serve as dest
                  // Change line attributes
                  d3.select(this)
                  .classed('hover', true)
                  .raise(); // Bring line forward

                  // Find the other endpoint of the line and change its color
                  const otherIata = sourceIata === d['iata'] ? destinationIata : sourceIata;
                  circles.filter(c => c['iata'] === otherIata)
                  .classed('connected', true)
                  .raise();

                }/*else{ // turn all other data less visible
                  d3.select(this)
                  .classed('hidden', true);
                };*/
              });
              d3.select(this).raise(); // Bring circle forward

              // Show circle info beside the mouse
              d3.select('#tooltip')
                .style('left', (event.pageX + 10) + 'px')
                .style('top', (event.pageY - 28) + 'px')
                .style('opacity', 1)
                .text(d['iata']); // Show the information for airport iata
          });

          // Add mouseout event listeners to circles
          circles.on('mouseout', function(event, d) {
            //circles.classed('hidden', false);

            const circle = d3.select(this);
            // Revert circle color to default
            circle.classed('hover', false); 
            // revert line color to default
            d3.selectAll('line').each(function() {
              d3.select(this).classed('hover', false); //.classed('hidden', false)
            });
            //revert all circles color to default
            circles.classed('connected', false);
            //circles.attr('fill', c => c === circle.node() ? 'red' : '#ff6600').attr('opacity', 0.9); // Revert other endpoint color
            
            // Hide circle info
            d3.select('#tooltip')
              .style('opacity', 0);
          });

          // Add click event listeners to circles
          circles.on('click', function(event, d){
            const circle = d3.select(this);
            if (circle.classed('active')) {
              // The circle is active, revert to original state
              circle.classed('active', false);
              d3.selectAll('line').each(function(lineData) {
                const sourceIata = lineData['Source iata'];
                const destinationIata = lineData['Dest iata'];
                if (destinationIata === d['iata'] || sourceIata === d['iata']) { // Find routes that serve as dest
                  d3.select(this).classed('active', false);
                }
              });
            } else {
              // The circle is not active, apply the active (hover-like) state
              circle.classed('active', true); 
              d3.selectAll('line').each(function(lineData) {
                const sourceIata = lineData['Source iata'];
                const destinationIata = lineData['Dest iata'];
                if (destinationIata === d['iata'] || sourceIata === d['iata']) { // Find routes that serve as dest
                  d3.select(this).classed('active', true);
                }
              });
            }
          });
        });

      })
      .catch(error => {
        console.error('Error loading map data:', error);
      });
  });

  // Function to handle search
  function handleSearch() {
    const searchTerm = document.getElementById('searchInput').value.trim().toUpperCase(); // Get search term
    if (searchTerm === '') {
      hideSuggestions();
      // If search term is empty, revert colors to default
      circles.attr('fill', '#444').attr('r', 1).attr('opacity', 1);
      d3.selectAll('line').attr('stroke', '#777777').attr('stroke-dasharray', '2 2').attr('stroke-opacity', 0.2).attr('stroke-width', 0.2);
      // Clear latitude and longitude
      document.getElementById('latLong').innerText = '';
      // Clear connected airports
      document.getElementById('connectedAirports').innerText = '';
      // Hide circle info
      d3.select('#tooltip')
        .style('opacity', 0);
    } else {
      // Filter airport codes that start with the search term
      suggestions = flattenedData.filter(d => d.iata.startsWith(searchTerm));
      // Show suggestions in the dropdown list
      showSuggestions();
      // If there's only one unique suggestion, hide suggestions and trigger search
      if (searchTerm.length === 2){
        circles.attr('fill', '#444').attr('r', 1).attr('opacity', 1);
        d3.selectAll('line').attr('stroke', '#777777').attr('stroke-dasharray', '2 2').attr('stroke-opacity', 0.2).attr('stroke-width', 0.2);
        // Clear latitude and longitude
        document.getElementById('latLong').innerText = '';
        // Clear connected airports
        document.getElementById('connectedAirports').innerText = '';
        // Hide circle info
        d3.select('#tooltip')
          .style('opacity', 0);  
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
    d3.selectAll('line').each(function(lineData) {
      const sourceIata = lineData['Source iata'];
      const destinationIata = lineData['Dest iata'];
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
    const airport = flattenedData.find(d => d.iata === searchTerm);
    if (airport) {
      document.getElementById('latLong').innerText = searchTerm + `: ${airport.airport} \n` + `Latitude: ${airport.lat}\n Longitude: ${airport.long}`;
    } else {
      document.getElementById('latLong').innerText = '';
    }
    // Show connected airports
    const connectedAirports = lines.reduce((connected, line) => {
      const sourceIata = line['Source iata'];
      const destinationIata = line['Dest iata'];
      if (sourceIata === searchTerm) connected.push(destinationIata);
      else if (destinationIata === searchTerm) connected.push(sourceIata);
      return connected;
    }, []);

    // Calculate distances to connected airports
    const distances = connectedAirports.map(airportIata => {
      const destinationAirport = flattenedData.find(airport => airport.iata === airportIata);
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
      const destinationAirport = flattenedData.find(airport => airport.iata === iata);
      if (destinationAirport) {
        const listItem = document.createElement('li');
        listItem.textContent = `${iata}: ${destinationAirport.airport} (${distance.toFixed(2)} km)`;
        connectedAirportsList.appendChild(listItem);
      }
    });

    // Show circle info for the searched airport
    const searchedAirport = flattenedData.find(d => d.iata === searchTerm);
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

  /*
  // Function to handle search
  function handleSearch() {
    const searchTerm = document.getElementById('searchInput').value.trim().toUpperCase(); // Get search term
    if (searchTerm === '') {
      // If search term is empty, revert colors to default
      circles.attr('fill', '#ff6600');
      lines.forEach(line => line.attr('stroke', 'orange'));
      // Clear latitude and longitude
      document.getElementById('latLong').innerText = '';
      // Clear connected airports
      document.getElementById('connectedAirports').innerText = '';
    } else {
      // Highlight corresponding airport and segment
      circles.attr('fill', d => d.iata === searchTerm ? 'red' : '#ff6600');
      lines.forEach(line => {
        const sourceIata = line.data()[0].source_iata;
        const destinationIata = line.data()[0].destination_iata;
        if (sourceIata === searchTerm || destinationIata === searchTerm) {
          line.attr('stroke', 'red');
          const otherIata = sourceIata === searchTerm ? destinationIata : sourceIata;
          circles.filter(c => c.iata === otherIata).attr('fill', 'red');
        }
      });
      // Display latitude and longitude
      const airport = circles.filter(d => d.iata === searchTerm).datum();
      if (airport) {
        document.getElementById('latLong').innerText = `Latitude: ${airport.lat}, Longitude: ${airport.long}`;
      } else {
        document.getElementById('latLong').innerText = '';
      }
      // Show connected airports
      const connectedAirports = lines.reduce((connected, line) => {
        const sourceIata = line.data()[0].source_iata;
        const destinationIata = line.data()[0].destination_iata;
        if (sourceIata === searchTerm) connected.push(destinationIata);
        else if (destinationIata === searchTerm) connected.push(sourceIata);
        return connected;
      }, []);
      document.getElementById('connectedAirports').innerText = connectedAirports.length > 0
        ? `Destinations: ${connectedAirports.join(', ')}`
        : '';
    }
  }*/
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
    opacity: 0;
  }

  /* Styling for search bar */
  #searchInput {
    padding: 10px;
    border-radius: 5px;
    border: 1px solid #ccc;
    font-size: 16px;
    outline: none;
    transition: border-color 0.3s ease-in-out;
    width: 200px; /* Adjust width as needed */
  }

  #searchInput:focus {
    border-color: #007bff; /* Change border color on focus */
  }

  /* Style for latitude and longitude display */
  #latLong {
    font-size: 14px;
    margin-top: 10px;
  }

  /* Style for connected airports display */
  #connectedAirports {
    font-size: 14px;
    margin-top: 10px;
  }
</style>

<div id="tooltip"></div>

<!-- Search bar -->
<div style="position: absolute; top: 20px; right: 20px;">
  <input type="text" id="searchInput" on:input={handleSearch} placeholder="Search airport...">
  <div id="searchLabel"></div> <!-- Suggestions will be displayed here -->
</div>

<!-- Latitude and Longitude -->
<div id="latLong" style="position: absolute; top: 50px; right: 20px;"></div>

<!-- Connected airports -->
<div id="connectedAirports" style="position: absolute; top: 70px; right: 20px;"></div>