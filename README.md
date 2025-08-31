# R.-Oasis-Route-planning-
<!DOCTYPE html>
<html lang="my">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Delivery Route Planner</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6;
            color: #1f2937;
        }
        .container-panel {
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1), 0 1px 3px rgba(0, 0, 0, 0.08);
        }
        #map {
            min-height: 400px;
        }
        @media (min-width: 768px) {
            .container-panel {
                height: 100vh;
                overflow-y: auto;
            }
        }
    </style>
</head>
<body class="flex flex-col md:flex-row h-screen">

    <!-- Control Panel -->
    <div class="flex flex-col p-6 bg-white container-panel md:w-1/3">
        <h1 class="text-3xl font-bold mb-2 text-center text-blue-600">
            <span class="inline-block align-middle mr-2">📍</span> Delivery လမ်းကြောင်းစီမံခန့်ခွဲသူ
        </h1>
        <p class="text-gray-600 mb-6 text-center text-sm">တည်နေရာများထည့်ပြီး အုပ်စုလိုက် လမ်းကြောင်းဆွဲပါ။</p>

        <!-- Current User Info -->
        <div class="bg-gray-100 rounded-lg p-3 text-sm mb-4">
            <p class="font-medium text-gray-700">အကောင့်: <span id="user-id" class="break-all font-mono text-gray-900">...</span></p>
        </div>

        <div class="mb-4">
            <label for="location-input" class="block text-sm font-medium text-gray-700 mb-1">လိပ်စာထည့်သွင်းပါ</label>
            <input type="text" id="location-input" placeholder="ဥပမာ: 320 W 34th St, New York, NY"
                   class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500">
        </div>

        <button id="add-location-btn" class="w-full bg-blue-600 text-white font-semibold py-3 px-4 rounded-lg shadow-md hover:bg-blue-700 transition-colors duration-200">
            တည်နေရာထည့်မည်
        </button>

        <!-- Message/Error Box -->
        <div id="message-box" class="mt-4 p-3 hidden rounded-lg text-sm" role="alert"></div>

        <div class="mt-6">
            <h2 class="text-xl font-bold mb-2">အုပ်စုများ</h2>
            <div id="groups-container" class="space-y-3">
                <!-- Groups will be rendered here -->
            </div>
            <button id="add-group-btn" class="w-full bg-green-500 text-white font-semibold py-2 px-4 rounded-lg shadow-md hover:bg-green-600 transition-colors duration-200 mt-4">
                အုပ်စုအသစ်ဖန်တီးမည်
            </button>
        </div>

        <div class="mt-6 pt-6 border-t">
            <h2 class="text-xl font-bold mb-2">လက်ရှိ အုပ်စုများ</h2>
            <ul id="locations-list" class="space-y-2">
                <!-- Locations in selected group will be rendered here -->
            </ul>
            <button id="plan-route-btn" class="w-full bg-purple-600 text-white font-semibold py-3 px-4 rounded-lg shadow-md hover:bg-purple-700 transition-colors duration-200 mt-4">
                လမ်းကြောင်းရှာရန်
            </button>
        </div>
    </div>

    <!-- Map Container -->
    <div id="map" class="w-full md:w-2/3 shadow-inner bg-gray-200"></div>

    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, onSnapshot, collection, query, setDoc, updateDoc, arrayUnion, arrayRemove } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Set Firebase debug logs
        setLogLevel('debug');

        // IMPORTANT: These are global variables provided by the environment. DO NOT change them.
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let db, auth, userId;
        let isAuthReady = false;

        // UI elements
        const locationInput = document.getElementById('location-input');
        const addLocationBtn = document.getElementById('add-location-btn');
        const addGroupBtn = document.getElementById('add-group-btn');
        const planRouteBtn = document.getElementById('plan-route-btn');
        const messageBox = document.getElementById('message-box');
        const groupsContainer = document.getElementById('groups-container');
        const locationsList = document.getElementById('locations-list');
        const userIdDisplay = document.getElementById('user-id');

        // Google Maps variables
        let map, directionsService, directionsRenderer;
        const GOOGLE_MAPS_API_KEY = "AIzaSyADPbWyTP29XK1gJvYrCj1hmwm_24dkIMg"; // Change this to your API key

        let groups = {};
        let activeGroupId = null;

        // Initialize Firebase
        const app = initializeApp(firebaseConfig);
        db = getFirestore(app);
        auth = getAuth(app);

        // Firebase Authentication State Listener
        onAuthStateChanged(auth, async (user) => {
            if (user) {
                userId = user.uid;
            } else {
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken).catch(e => console.error("Custom token sign-in failed:", e));
                } else {
                    await signInAnonymously(auth).catch(e => console.error("Anonymous sign-in failed:", e));
                }
                userId = auth.currentUser?.uid || 'anonymous-user';
            }
            isAuthReady = true;
            userIdDisplay.textContent = userId;
            setupFirestoreListener();
        });

        // Initialize Google Maps
        window.initMap = function() {
            directionsService = new google.maps.DirectionsService();
            directionsRenderer = new google.maps.DirectionsRenderer({ suppressMarkers: false });
            const myanmar = { lat: 16.8409, lng: 96.1735 };
            map = new google.maps.Map(document.getElementById("map"), {
                zoom: 12,
                center: myanmar,
            });
            directionsRenderer.setMap(map);
        };

        // Load Google Maps API Script
        const script = document.createElement('script');
        script.src = `https://maps.googleapis.com/maps/api/js?key=${GOOGLE_MAPS_API_KEY}&callback=initMap`;
        script.async = true;
        document.head.appendChild(script);

        // Firestore Listener for real-time updates
        function setupFirestoreListener() {
            if (!isAuthReady || !userId) {
                console.log("Firestore listener not set up. Auth not ready.");
                return;
            }

            const groupsCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/deliveryLocations`);
            onSnapshot(groupsCollectionRef, (snapshot) => {
                groups = {};
                snapshot.forEach(doc => {
                    const groupData = doc.data();
                    groups[doc.id] = groupData;
                });
                renderGroups();
                updateMapWithData();
            }, (error) => {
                console.error("Error listening to Firestore:", error);
                showMessage("ဒေတာများ load လုပ်ရာတွင် အမှားအယွင်းရှိခဲ့သည်။", 'bg-red-100 text-red-800');
            });
        }

        // --- Core Functions ---

        async function addLocationToGroup() {
            if (!activeGroupId) {
                showMessage("တည်နေရာထည့်ဖို့အတွက် အုပ်စုတစ်ခုအရင်ရွေးပါ။", 'bg-yellow-100 text-yellow-800');
                return;
            }

            const address = locationInput.value.trim();
            if (!address) {
                showMessage("လိပ်စာထည့်သွင်းပေးပါ။", 'bg-red-100 text-red-800');
                return;
            }

            showMessage("တည်နေရာရှာဖွေနေပါသည်...", 'bg-blue-100 text-blue-800');
            
            try {
                // Use Geocoder to get lat/lng from address
                const geocoder = new google.maps.Geocoder();
                const response = await geocoder.geocode({ address: address });
                if (response.results.length === 0) {
                    showMessage("လိပ်စာရှာမတွေ့ပါ။", 'bg-red-100 text-red-800');
                    return;
                }

                const location = response.results[0].geometry.location;
                const newLocation = {
                    address: response.results[0].formatted_address,
                    lat: location.lat(),
                    lng: location.lng(),
                    timestamp: new Date().toISOString()
                };

                const docRef = doc(db, `artifacts/${appId}/users/${userId}/deliveryLocations`, activeGroupId);
                await updateDoc(docRef, {
                    locations: arrayUnion(newLocation)
                });

                locationInput.value = '';
                showMessage("တည်နေရာကို အောင်မြင်စွာထည့်သွင်းပြီးပါပြီ။", 'bg-green-100 text-green-800');
            } catch (error) {
                console.error("Error adding location:", error);
                showMessage("တည်နေရာထည့်ရာတွင် အမှားအယွင်းရှိခဲ့သည်။", 'bg-red-100 text-red-800');
            }
        }

        async function createNewGroup() {
            const groupName = prompt("အုပ်စုအမည်တစ်ခု ပေးပါ။");
            if (groupName) {
                const docRef = doc(db, `artifacts/${appId}/users/${userId}/deliveryLocations`, Date.now().toString());
                await setDoc(docRef, { name: groupName, locations: [] });
                showMessage(`'${groupName}' အုပ်စုကို ဖန်တီးပြီးပါပြီ။`, 'bg-green-100 text-green-800');
            }
        }

        function planRoute() {
            if (!activeGroupId) {
                showMessage("လမ်းကြောင်းဆွဲရန်အတွက် အုပ်စုတစ်ခုရွေးပါ။", 'bg-yellow-100 text-yellow-800');
                return;
            }
            const group = groups[activeGroupId];
            if (!group || group.locations.length < 2) {
                showMessage("လမ်းကြောင်းဆွဲရန် အနည်းဆုံးတည်နေရာ ၂ ခု လိုအပ်သည်။", 'bg-yellow-100 text-yellow-800');
                return;
            }

            // Waypoints for the route
            const waypoints = group.locations.slice(1, -1).map(loc => ({
                location: { lat: loc.lat, lng: loc.lng },
                stopover: true
            }));

            directionsService.route(
                {
                    origin: { lat: group.locations[0].lat, lng: group.locations[0].lng },
                    destination: { lat: group.locations[group.locations.length - 1].lat, lng: group.locations[group.locations.length - 1].lng },
                    waypoints: waypoints,
                    optimizeWaypoints: true, // This is the new feature! It finds the most efficient order for the waypoints.
                    travelMode: 'DRIVING'
                },
                (response, status) => {
                    if (status === "OK") {
                        directionsRenderer.setDirections(response);
                        showMessage("လမ်းကြောင်းကို အောင်မြင်စွာ တွက်ချက်ပြီးပါပြီ။", 'bg-green-100 text-green-800');
                    } else {
                        showMessage("လမ်းကြောင်းရှာမတွေ့ပါ။ အချက်အလက်များကို ပြန်စစ်ဆေးပါ။", 'bg-red-100 text-red-800');
                        console.error("Directions request failed due to " + status);
                    }
                }
            );
        }

        // --- UI Rendering and Event Handling ---

        function renderGroups() {
            groupsContainer.innerHTML = '';
            locationsList.innerHTML = '';
            Object.keys(groups).forEach(id => {
                const group = groups[id];
                const button = document.createElement('button');
                button.textContent = group.name;
                button.className = `w-full text-left px-4 py-2 rounded-lg transition-colors duration-200 ${id === activeGroupId ? 'bg-blue-200 text-blue-800' : 'bg-gray-200 text-gray-700 hover:bg-gray-300'}`;
                button.onclick = () => {
                    activeGroupId = id;
                    renderGroups(); // Re-render to show active state
                    renderLocationsInGroup(group.locations);
                    updateMapWithData();
                };
                groupsContainer.appendChild(button);
            });
            if (activeGroupId && groups[activeGroupId]) {
                renderLocationsInGroup(groups[activeGroupId].locations);
            }
        }

        function renderLocationsInGroup(locations) {
            locationsList.innerHTML = '';
            if (locations.length === 0) {
                locationsList.innerHTML = '<li class="text-gray-500">ဤအုပ်စုတွင် တည်နေရာများ မရှိသေးပါ။</li>';
                return;
            }
            locations.forEach((loc, index) => {
                const li = document.createElement('li');
                li.className = 'bg-gray-100 rounded-lg p-2 text-sm flex items-center justify-between';
                li.innerHTML = `
                    <span>${index + 1}. ${loc.address}</span>
                    <button class="text-red-500 hover:text-red-700" onclick="removeLocation('${loc.address}')">
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                            <path fill-rule="evenodd" d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm6 0a1 1 0 112 0v6a1 1 0 11-2 0V8z" clip-rule="evenodd" />
                        </svg>
                    </button>
                `;
                locationsList.appendChild(li);
            });
        }

        window.removeLocation = async function(addressToRemove) {
            if (!activeGroupId) return;

            const group = groups[activeGroupId];
            const updatedLocations = group.locations.filter(loc => loc.address !== addressToRemove);

            const docRef = doc(db, `artifacts/${appId}/users/${userId}/deliveryLocations`, activeGroupId);
            await updateDoc(docRef, { locations: updatedLocations });
            showMessage("တည်နေရာကိုဖျက်လိုက်ပါပြီ။", 'bg-green-100 text-green-800');
        };

        function updateMapWithData() {
            if (!map || !activeGroupId || !groups[activeGroupId]) return;

            const group = groups[activeGroupId];
            const locations = group.locations;
            
            // Clear previous route
            directionsRenderer.setDirections({ routes: [] });

            // Clear previous markers
            const markers = [];
            const bounds = new google.maps.LatLngBounds();

            locations.forEach((loc, index) => {
                const position = new google.maps.LatLng(loc.lat, loc.lng);
                const marker = new google.maps.Marker({
                    position: position,
                    map: map,
                    label: `${index + 1}`,
                    title: loc.address
                });
                markers.push(marker);
                bounds.extend(position);
            });

            if (locations.length > 0) {
                map.fitBounds(bounds);
            }
        }
        
        function showMessage(text, classes) {
            messageBox.textContent = text;
            messageBox.className = `mt-4 p-3 rounded-lg text-sm ${classes}`;
            messageBox.style.display = 'block';
            setTimeout(() => {
                messageBox.style.display = 'none';
            }, 5000);
        }

        // Event Listeners
        addLocationBtn.addEventListener('click', addLocationToGroup);
        addGroupBtn.addEventListener('click', createNewGroup);
        planRouteBtn.addEventListener('click', planRoute);

    </script>
</body>
</html>

