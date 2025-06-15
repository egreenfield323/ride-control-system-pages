<template>
  <div class="main-container">
    <!-- Vehicle Tracking Section -->
    <div class="vehicle-tracking-section">
      <h1 class="vehicle-tracking-header">🎢 Vehicle Tracking</h1>

      <div class="vehicle-legend">
        <div class="legend-item">
          <div class="legend-dot" style="background: #4caf50;"></div>
          <span>Block Zone Occupied</span>
        </div>
        <div class="legend-item">
          <div class="legend-dot" style="background: #2196f3;"></div>
          <span>Loading Dock</span>
        </div>
        <div class="legend-item">
          <div class="legend-dot" style="background: rgba(255, 255, 255, 0.2);"></div>
          <span>Empty Zone</span>
        </div>
      </div>

      <div class="block-zones">
        <div v-for="zone in blockZones" :key="zone.id" :class="['block-zone', {
          'occupied': zone.vehicles.length > 0,
          'loading-dock': zone.id === 'loading'
        }]">
          <div class="block-zone-title">{{ zone.name }}</div>
          <div class="vehicle-queue">
            <div v-for="vehicle in zone.vehicles" :key="vehicle.id" class="vehicle">
              {{ vehicle.name }}
            </div>
          </div>
          <div v-if="zone.vehicles.length === 0" style="color: #666; font-size: 12px;">
            Empty
          </div>
        </div>
      </div>

      <div class="dispatch-controls">
        <button class="btn-dispatch" @click="dispatchVehicle" :disabled="!canDispatch">
          🚀 Dispatch Next Vehicle
        </button>
        <button class="btn-dispatch" @click="addVehicleToLoading" :disabled="totalVehicles >= maxVehicles"
          style="background: linear-gradient(45deg, #9c27b0, #ba68c8);">
          ➕ Add Vehicle to Loading
        </button>
      </div>
    </div>

    <div class="control-panel">
      <!-- Ride Operations Section -->
      <div class="section">
        <h2>🎢 Ride Operations</h2>

        <div :class="['ride-status', rideStatusClass]">
          <span :class="['status-indicator', rideStatusIndicator]"></span>
          {{ rideStatus }}
        </div>

        <button class="control-button btn-start" @click="startRide"
          :disabled="rideStatus === 'RUNNING' || emergencyStop">
          Start Ride
        </button>

        <button class="control-button btn-stop" @click="stopRide" :disabled="rideStatus === 'STOPPED'">
          Stop Ride
        </button>

        <button class="control-button btn-emergency" @click="emergencyStopRide">
          🚨 EMERGENCY STOP
        </button>

        <button class="control-button btn-maintenance" @click="toggleMaintenance">
          {{ maintenanceMode ? 'Exit Maintenance' : 'Enter Maintenance' }}
        </button>

        <button class="control-button btn-reset" @click="resetSystem" :disabled="!emergencyStop && !maintenanceMode">
          Reset System
        </button>

        <div class="system-metrics">
          <div class="metric">
            <div class="metric-value">{{ cycleCount }}</div>
            <div class="metric-label">Cycles Today</div>
          </div>
          <div class="metric">
            <div class="metric-value">{{ guestsServed }}</div>
            <div class="metric-label">Guests Served</div>
          </div>
        </div>
      </div>

      <!-- Safety & Monitoring Section -->
      <div class="section">
        <h2>🛡️ Safety & Monitoring</h2>

        <h3 style="color: #4fc3f7; margin: 15px 0 10px 0;">Safety Systems</h3>
        <div class="safety-check" v-for="check in safetyChecks" :key="check.name">
          <span>{{ check.name }}</span>
          <span :class="['status-indicator', check.status ? 'status-operational' : 'status-critical']"></span>
        </div>

        <h3 style="color: #4fc3f7; margin: 15px 0 10px 0;">Weather Conditions</h3>
        <div class="weather-info">
          <div class="weather-temp">{{ weather.temperature.toFixed(2) }}°F</div>
          <div>{{ weather.condition }}</div>
          <div style="font-size: 12px; color: #9e9e9e;">
            Wind: {{ weather.windSpeed.toFixed(2) }} mph | Humidity: {{ weather.humidity.toFixed(2) }}%
          </div>
        </div>

        <h3 style="color: #4fc3f7; margin: 15px 0 10px 0;">System Health</h3>
        <div class="safety-check" v-for="system in systemHealth" :key="system.name">
          <span>{{ system.name }}</span>
          <span>{{ system.value }}</span>
        </div>
      </div>

      <!-- Queue & Capacity Section -->
      <div class="section">
        <h2>👥 Queue & Capacity</h2>

        <div class="queue-info">
          <span>Current Wait Time:</span>
          <span style="color: #4fc3f7; font-weight: bold;">{{ waitTime }} min</span>
        </div>

        <div class="queue-info">
          <span>Queue Length:</span>
          <span style="color: #4fc3f7; font-weight: bold;">{{ queueLength }} guests</span>
        </div>

        <div class="queue-info">
          <span>Active Vehicles:</span>
          <span style="color: #4fc3f7; font-weight: bold;">{{ totalVehicles }}/{{ maxVehicles }}</span>
        </div>

        <div class="queue-info">
          <span>Hourly Capacity:</span>
          <span style="color: #4fc3f7; font-weight: bold;">{{ hourlyCapacity }} guests/hr</span>
        </div>

        <h3 style="color: #4fc3f7; margin: 15px 0 10px 0;">Capacity Utilization</h3>
        <div class="capacity-bar">
          <div class="capacity-fill" :style="{ width: capacityPercent + '%' }"></div>
        </div>
        <div style="text-align: center; margin-top: 5px;">{{ capacityPercent }}%</div>

        <h3 style="color: #4fc3f7; margin: 15px 0 10px 0;">Activity Log</h3>
        <div style="max-height: 200px; overflow-y: auto;">
          <div class="log-entry" v-for="log in activityLog" :key="log.id">
            <div>{{ log.message }}</div>
            <div class="log-timestamp">{{ log.timestamp }}</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'RideControlSystem',
  data() {
    return {
      rideStatus: 'STOPPED',
      emergencyStop: false,
      maintenanceMode: false,
      rainAlertShown: false,
      cycleCount: 247,
      guestsServed: 2964,
      waitTime: 35,
      queueLength: 142,
      capacityPercent: 78,
      maxVehicles: 8,
      vehicleCounter: 3,
      passengersPerVehicle: 6,
      lastDispatchTimestamps: [],
      blockZones: [
        {
          id: 'loading',
          name: 'Loading Dock',
          vehicles: [
            { id: 1, name: 'Car A-01' },
            { id: 2, name: 'Car A-02' },
            { id: 3, name: 'Car A-03' }
          ],
          maxCapacity: 5
        },
        {
          id: 'lift1',
          name: 'Lift Hill Block',
          vehicles: [],
          maxCapacity: 1
        },
        {
          id: 'zone1',
          name: 'Cave Section',
          vehicles: [],
          maxCapacity: 1
        },
        {
          id: 'zone2',
          name: 'Drop Section',
          vehicles: [],
          maxCapacity: 1
        },
        {
          id: 'zone3',
          name: 'High Speed Turn',
          vehicles: [],
          maxCapacity: 1
        },
        {
          id: 'zone4',
          name: 'Final Drop',
          vehicles: [],
          maxCapacity: 1
        },
        {
          id: 'brake',
          name: 'Brake Run',
          vehicles: [],
          maxCapacity: 2
        },
        {
          id: 'unload',
          name: 'Unload Station',
          vehicles: [],
          maxCapacity: 3
        }
      ],
      safetyChecks: [
        { name: 'Block Zone System', status: true },
        { name: 'Emergency Brakes', status: true },
        { name: 'Restraint Systems', status: true },
        { name: 'Height Sensors', status: true },
        { name: 'Station Platforms', status: true },
        { name: 'Fire Suppression', status: true }
      ],
      weather: {
        temperature: 68,
        condition: 'Partly Cloudy',
        windSpeed: 8,
        humidity: 65
      },
      systemHealth: [
        { name: 'Motor Temperature', value: '165°F' },
        { name: 'Hydraulic Pressure', value: '2400 PSI' },
        { name: 'Power Consumption', value: '89 kW' },
        { name: 'Vibration Level', value: 'Normal' }
      ],
      activityLog: [
        { id: 1, message: 'System initialized - Ready for operation', timestamp: '14:23:15' }
      ],
      logCounter: 2,
      vehicleMovementInterval: null
    }
  },
  computed: {
    rideStatusClass() {
      switch (this.rideStatus) {
        case 'RUNNING': return 'status-running';
        case 'STOPPED': return 'status-stopped';
        case 'MAINTENANCE': return 'status-maintenance-mode';
        default: return 'status-stopped';
      }
    },
    rideStatusIndicator() {
      switch (this.rideStatus) {
        case 'RUNNING': return 'status-operational';
        case 'STOPPED': return this.emergencyStop ? 'status-critical' : 'status-warning';
        case 'MAINTENANCE': return 'status-maintenance';
        default: return 'status-warning';
      }
    },
    canDispatch() {
      const loadingDock = this.blockZones.find(z => z.id === 'loading');
      const liftHill = this.blockZones.find(z => z.id === 'lift1');
      return this.rideStatus === 'RUNNING' &&
        loadingDock.vehicles.length > 0 &&
        liftHill.vehicles.length === 0 &&
        !this.emergencyStop;
    },
    totalVehicles() {
      return this.blockZones.reduce((total, zone) => total + zone.vehicles.length, 0);
    },
    hourlyCapacity() {
      return Math.round(this.guestsServed / 60);
    },
    waitTime() {
      const now = Date.now();
      const windowMinutes = 5;
      const windowStart = now - windowMinutes * 60 * 1000;

      // Get recent dispatches
      const recentTimestamps = this.lastDispatchTimestamps.filter(ts => ts >= windowStart);
      const minutesInWindow = windowMinutes;

      let vehiclesPerMinute;

      if (recentTimestamps.length >= 1) {
        vehiclesPerMinute = recentTimestamps.length / minutesInWindow;
      } else {
        // fallback dispatch rate if no recent data
        vehiclesPerMinute = 1 / 2; // 1 vehicle every 2 minutes
      }

      // avoid unrealistic/skewed spikes
      vehiclesPerMinute = Math.max(0.1, Math.min(vehiclesPerMinute, 2));

      const guestsPerMinute = vehiclesPerMinute * this.passengersPerVehicle;
      const estimatedMinutes = Math.ceil(this.queueLength / guestsPerMinute);

      return Math.max(1, estimatedMinutes);
    }
  },
  methods: {
    startRide() {
      if (this.emergencyStop || this.maintenanceMode) return;

      this.rideStatus = 'RUNNING';
      this.addLog('Ride started - All systems operational');
      this.startVehicleMovement();
    },
    stopRide() {
      this.rideStatus = 'STOPPED';
      this.addLog('Ride stopped by operator');
      this.stopVehicleMovement();
    },
    emergencyStopRide() {
      this.rideStatus = 'STOPPED';
      this.emergencyStop = true;
      this.addLog('🚨 EMERGENCY STOP ACTIVATED - All vehicles halted');
      this.stopVehicleMovement();
      this.safetyChecks[1].status = false;
    },
    toggleMaintenance() {
      this.maintenanceMode = !this.maintenanceMode;
      if (this.maintenanceMode) {
        this.rideStatus = 'MAINTENANCE';
        this.addLog('Maintenance mode activated');
        this.stopVehicleMovement();
      } else {
        this.rideStatus = 'STOPPED';
        this.addLog('Maintenance mode deactivated');
      }
    },
    resetSystem() {
      this.emergencyStop = false;
      this.maintenanceMode = false;
      this.rideStatus = 'STOPPED';

      // reset safety systems
      this.safetyChecks.forEach(check => check.status = true);

      this.addLog('System reset complete - All systems running');
    },
    dispatchVehicle() {
      if (!this.canDispatch) return;

      const loadingDock = this.blockZones.find(z => z.id === 'loading');
      const liftHill = this.blockZones.find(z => z.id === 'lift1');

      if (loadingDock.vehicles.length > 0) {
        const vehicle = loadingDock.vehicles.shift();
        liftHill.vehicles.push(vehicle);

        const guestsOnVehicle = Math.min(this.queueLength, this.passengersPerVehicle);
        this.queueLength = Math.max(0, this.queueLength - guestsOnVehicle);
        this.guestsServed += guestsOnVehicle;

        this.lastDispatchTimestamps.push(Date.now());
        this.cleanOldDispatches();

        this.addLog(`Vehicle ${vehicle.name} dispatched with ${guestsOnVehicle} guests`);

        this.cycleCount++;
        this.capacityPercent = Math.min(100, Math.floor((this.guestsServed / 50)));
      }
    },
    cleanOldDispatches() {
      const oneHourAgo = Date.now() - 3600000;
      this.lastDispatchTimestamps = this.lastDispatchTimestamps.filter(ts => ts >= oneHourAgo);
    },
    addVehicleToLoading() {
      if (this.totalVehicles >= this.maxVehicles) return;

      const loadingDock = this.blockZones.find(z => z.id === 'loading');
      if (loadingDock.vehicles.length < loadingDock.maxCapacity) {
        this.vehicleCounter++;
        const newVehicle = {
          id: this.vehicleCounter,
          name: `Car A-${String(this.vehicleCounter).padStart(2, '0')}`
        };
        loadingDock.vehicles.push(newVehicle);
        this.addLog(`Vehicle ${newVehicle.name} added to loading dock`);
      }
    },
    startVehicleMovement() {
      this.stopVehicleMovement(); // clear any existing interval
      this.vehicleMovementInterval = setInterval(() => {
        this.moveVehiclesThroughZones();
      }, 4000); // move vehicle every 4 seconds
    },
    stopVehicleMovement() {
      if (this.vehicleMovementInterval) {
        clearInterval(this.vehicleMovementInterval);
        this.vehicleMovementInterval = null;
      }
    },
    moveVehiclesThroughZones() {
      if (this.rideStatus !== 'RUNNING' || this.emergencyStop) return;

      // move vehicles through zones in reverse order to avoid conflicts
      const zoneOrder = ['unload', 'brake', 'zone4', 'zone3', 'zone2', 'zone1', 'lift1'];

      zoneOrder.forEach(zoneId => {
        const currentZone = this.blockZones.find(z => z.id === zoneId);
        if (currentZone.vehicles.length === 0) return;

        let nextZone;
        if (zoneId === 'unload') {
          // vehicle has finished, goes back to loading
          nextZone = this.blockZones.find(z => z.id === 'loading');
        } else {
          // find next zone in sequence
          const currentIndex = this.blockZones.findIndex(z => z.id === zoneId);
          const nextIndex = (currentIndex + 1) % this.blockZones.length;
          nextZone = this.blockZones[nextIndex];
        }

        // check if next zone has capacity
        if (nextZone.vehicles.length < nextZone.maxCapacity) {
          const vehicle = currentZone.vehicles.shift();
          nextZone.vehicles.push(vehicle);

          if (zoneId === 'unload') {
            this.addLog(`Vehicle ${vehicle.name} completed circuit, returned to loading`);
          } else {
            this.addLog(`Vehicle ${vehicle.name} moved from ${currentZone.name} to ${nextZone.name}`);
          }
        }
      });
    },
    addLog(message) {
      const now = new Date();
      const timestamp = now.toTimeString().substr(0, 8);

      this.activityLog.unshift({
        id: this.logCounter++,
        message: message,
        timestamp: timestamp
      });

      // keep only last 20 entries
      if (this.activityLog.length > 20) {
        this.activityLog.pop();
      }
    },
    updateWeather() {
      // simulate weather changes
      this.weather.temperature += (Math.random() - 0.5) * 2;
      this.weather.windSpeed += (Math.random() - 0.5) * 3;
      this.weather.humidity += (Math.random() - 0.5) * 5;

      // simulate rain
      if (Math.random() < 0.05) { // 5% chance of rain
        this.weather.condition = 'Rain';
        if (!this.rainAlertShown) {
          this.rainAlertShown = true;
          this.addLog('⚠️ Weather alert: Rain detected. Prepare to shut down the ride.');
        }
      } else {
        this.rainAlertShown = false;
        this.weather.condition = Math.random() < 0.5 ? 'Partly Cloudy' : 'Clear';
      }
    }

  },
  mounted() {
    // real-time updates
    setInterval(() => {
      if (!this.emergencyStop && !this.maintenanceMode) {
        this.updateWeather();

        // simulate new guests entering queue
        const newGuests = Math.floor(Math.random() * 4) + 1; // 1–4 new guests per update
        this.queueLength += newGuests;

        // randomly update system health (testing)
        if (Math.random() < 0.1) {
          const randomSystem = this.systemHealth[Math.floor(Math.random() * this.systemHealth.length)];
          if (randomSystem.name === 'Motor Temperature') {
            randomSystem.value = Math.floor(160 + Math.random() * 20) + '°F';
          }
        }
      }
    }, 5000); // queue increases every 5 seconds

    // initial log entry
    this.addLog('Control system initialized - Ready for operation');
  },
  beforeUnmount() {
    this.stopVehicleMovement();
  }
}
</script>