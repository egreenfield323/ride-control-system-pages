<template>
  <div class="container-fluid py-3">
    <div class="card mb-3">
      <div class="card-header">Vehicle Tracking</div>
      <div class="card-body">
        <div class="d-flex gap-3 small mb-2">
          <span><span class="badge text-bg-success">&nbsp;</span> Occupied</span>
          <span><span class="badge text-bg-primary">&nbsp;</span> Loading dock</span>
          <span><span class="badge text-bg-light border">&nbsp;</span> Empty</span>
        </div>

        <div class="row row-cols-2 row-cols-md-4 g-2 mb-3">
          <div class="col" v-for="zone in blockZones" :key="zone.id">
            <div :class="['card h-100', {
              'border-success': zone.vehicles.length > 0,
              'border-primary': zone.id === 'loading'
            }]">
              <div class="card-body p-2">
                <div class="fw-bold">{{ zone.name }}</div>
                <div class="small text-muted mb-1">{{ zone.vehicles.length }}/{{ zone.maxCapacity }}</div>
                <div class="font-monospace small" v-for="vehicle in zone.vehicles" :key="vehicle.id">{{ vehicle.name }}</div>
              </div>
            </div>
          </div>
        </div>

        <button class="btn btn-primary me-2" @click="dispatchVehicle" :disabled="!canDispatch">Dispatch next vehicle</button>
        <button class="btn btn-secondary" @click="addVehicleToLoading" :disabled="totalVehicles >= maxVehicles">Add vehicle to loading</button>
      </div>
    </div>

    <div class="row g-3">
      <div class="col-lg-4">
        <div class="card h-100">
          <div class="card-header">Ride Operations</div>
          <div class="card-body">
            <div :class="['alert py-2', rideStatusClass]">Status: {{ rideStatus }}</div>

            <div class="d-grid gap-2 mb-3">
              <button class="btn btn-success" @click="startRide" :disabled="rideStatus === 'RUNNING' || emergencyStop">Start</button>
              <button class="btn btn-secondary" @click="stopRide" :disabled="rideStatus === 'STOPPED'">Stop</button>
              <button class="btn btn-danger" @click="emergencyStopRide">Emergency stop</button>
              <button class="btn btn-outline-secondary" @click="toggleMaintenance">
                {{ maintenanceMode ? 'Exit maintenance' : 'Enter maintenance' }}
              </button>
              <button class="btn btn-outline-secondary" @click="resetSystem" :disabled="!emergencyStop && !maintenanceMode">Reset system</button>
            </div>

            <table class="table table-sm mb-0">
              <tbody>
                <tr><td>Cycles today</td><td class="text-end">{{ cycleCount }}</td></tr>
                <tr><td>Guests served</td><td class="text-end">{{ guestsServed }}</td></tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>

      <div class="col-lg-4">
        <div class="card h-100">
          <div class="card-header">Safety</div>
          <div class="card-body">
            <h6>Safety systems</h6>
            <table class="table table-sm">
              <tbody>
                <tr v-for="check in safetyChecks" :key="check.name">
                  <td>{{ check.name }}</td>
                  <td :class="['text-end', check.status ? 'text-success' : 'text-danger fw-bold']">{{ check.status ? 'OK' : 'FAULT' }}</td>
                </tr>
              </tbody>
            </table>

            <h6>Weather</h6>
            <table class="table table-sm mb-0">
              <tbody>
                <tr><td>Condition</td><td class="text-end">{{ weather.condition }}</td></tr>
                <tr><td>Wind</td><td class="text-end">{{ weather.windSpeed.toFixed(1) }} mph</td></tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>

      <div class="col-lg-4">
        <div class="card h-100">
          <div class="card-header">Queue and Capacity</div>
          <div class="card-body">
            <table class="table table-sm">
              <tbody>
                <tr><td>Wait time</td><td class="text-end">{{ waitTime }} min</td></tr>
                <tr><td>Queue length</td><td class="text-end">{{ queueLength }} guests</td></tr>
                <tr><td>Vehicles in service</td><td class="text-end">{{ totalVehicles }}/{{ maxVehicles }}</td></tr>
                <tr><td>Hourly capacity</td><td class="text-end">{{ hourlyCapacity }} guests/hr</td></tr>
                <tr><td>Capacity utilization</td><td class="text-end">{{ capacityPercent }}%</td></tr>
              </tbody>
            </table>

            <h6>Activity log</h6>
            <ul class="list-group list-group-flush small activity-log">
              <li class="list-group-item px-0 py-1" v-for="log in activityLog" :key="log.id">
                <span class="text-muted">{{ log.timestamp }}</span> {{ log.message }}
              </li>
            </ul>
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
        { name: 'Restraint Systems', status: true }
      ],
      weather: {
        condition: 'Clear',
        windSpeed: 8
      },
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
        case 'RUNNING': return 'alert-success';
        case 'STOPPED': return 'alert-danger';
        case 'MAINTENANCE': return 'alert-warning';
        default: return 'alert-danger';
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
      this.addLog('EMERGENCY STOP ACTIVATED - All vehicles halted');
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
      this.weather.windSpeed += (Math.random() - 0.5) * 3;

      // simulate rain
      if (Math.random() < 0.05) { // 5% chance of rain
        this.weather.condition = 'Rain';
        if (!this.rainAlertShown) {
          this.rainAlertShown = true;
          this.addLog('Weather alert: Rain detected. Prepare to shut down the ride.');
        }
      } else {
        this.rainAlertShown = false;
        this.weather.condition = 'Clear';
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