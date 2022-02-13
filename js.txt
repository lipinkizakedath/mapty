'use strict';

// prettier-ignore

const form = document.querySelector('.form');
const containerWorkouts = document.querySelector('.workouts');
const inputType = document.querySelector('.form__input--type');
const inputDistance = document.querySelector('.form__input--distance');
const inputDuration = document.querySelector('.form__input--duration');
const inputCadence = document.querySelector('.form__input--cadence');
const inputElevation = document.querySelector('.form__input--elevation');

class Workout {
  date = new Date();
  id = (Date.now() + '').slice(-10);
  clicks = 0;

  constructor(coords, distance, duration) {
    this.coords = coords;
    this.distance = distance; // In kilometers
    this.duration = duration; // in Minutes
  }

  _setDiscription() {
    const months = [
      'January',
      'February',
      'March',
      'April',
      'May',
      'June',
      'July',
      'August',
      'September',
      'October',
      'November',
      'December',
    ];

    this.discription = `${this.type[0].toUpperCase()}${this.type.slice(1)} on ${
      months[this.date.getMonth()]
    } ${this.date.getDate()}`;
  }
}

class Running extends Workout {
  type = 'running';
  constructor(coords, distance, duration, cadence) {
    super(coords, distance, duration);
    this.cadence = cadence;
    this.calculatePace();
    this._setDiscription();
  }

  calculatePace() {
    this.pace = this.duration / this.distance;
    return this.pace;
  }
}

class Cycling extends Workout {
  type = 'cycling';
  constructor(coords, distance, duration, elevationGain) {
    super(coords, distance, duration);
    this.elevationGain = elevationGain;
    this.calculateSpeed();
    this._setDiscription();
  }

  calculateSpeed() {
    this.speed = this.distance / (this.duration / 60);
    return this.speed;
  }
}

class App {
  // Private fields
  #map;
  #mapEvent;
  #workout = [];
  #mapZoomLevel = 14;

  // Constructor
  constructor() {
    // Get user's position
    this._getPosition();

    // Get localstorage data
    this._getFromLocalStorage();

    // Form submission
    form.addEventListener('submit', this._newWorkOut.bind(this));
    inputType.addEventListener('change', this._toggleElevationField);
    containerWorkouts.addEventListener('click', this._moveToPopup.bind(this));
  }

  // Get the currenet geolocation of the user
  _getPosition() {
    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition(
        this._loadMap.bind(this),
        error => {
          alert(error.message);
        }
      );
    }
  }

  // Load the map based on the current geolocation
  _loadMap({ coords }) {
    this.#map = L.map('map').setView(
      [coords.latitude, coords.longitude],
      this.#mapZoomLevel
    );
    L.tileLayer('https://{s}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png', {
      attribution:
        '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
    }).addTo(this.#map);

    this.#map.on('click', this._showForm.bind(this));

    // Loops all the workouts from the array and add the marker and listing
    this.#workout.forEach(el => {
      this._renderWorkout(el);
      this._renderWorkoutMarker(el);
    });
  }

  // Show the form to add the data
  _showForm(event) {
    this.#mapEvent = event;
    form.classList.remove('hidden');
    inputDistance.focus();
  }

  // Hide input form section
  _hideForm() {
    inputDistance.value =
      inputDuration.value =
      inputCadence.value =
      inputElevation.value =
        '';
    form.style.display = 'none';
    form.classList.add('hidden');
    setTimeout(() => {
      form.style.display = 'grid';
    }, 2000);
  }

  // Toggle Elevation field while changing the workout type
  _toggleElevationField() {
    inputElevation.closest('.form__row').classList.toggle('form__row--hidden');
    inputCadence.closest('.form__row').classList.toggle('form__row--hidden');
  }

  // Adding new workout to the list
  _newWorkOut(e) {
    // Healper method to validate the number inputs
    const validInput = (...inputs) => inputs.every(val => Number.isFinite(val));

    // Healper method to validate the number is positive
    const checkPositive = (...input) => input.every(val => val > 0);

    e.preventDefault();

    // Get data from the form
    const type = inputType.value;
    const distance = +inputDistance.value;
    const duration = +inputDuration.value;
    const { lat, lng } = this.#mapEvent.latlng;
    let workout;

    // Check the data is valid or not
    // If workout running, create running object
    //
    if (type === 'running') {
      const cadence = +inputCadence.value;
      if (
        !validInput(distance, duration, cadence) ||
        !checkPositive(distance, duration, cadence)
      )
        return alert('Input should be a positive number');

      workout = new Running([lat, lng], distance, duration, cadence);
    }

    //
    // If workout cycling, create cycling object
    //
    if (type === 'cycling') {
      const elevation = +inputElevation.value;
      if (
        !validInput(distance, duration, elevation) ||
        !checkPositive(distance, duration)
      )
        return alert('Input should be a positive number');

      workout = new Cycling([lat, lng], distance, duration, elevation);
    }

    // Add new object to workout array
    this.#workout.push(workout);

    // Render workout on map as Marker
    this._renderWorkoutMarker(workout);

    // Render workout on list
    this._renderWorkout(workout);

    // Hide input fields and clear the values
    this._hideForm();

    // Setting the data to localstorage
    this._setLocalStorage();
  }

  // Render marker on the map
  _renderWorkoutMarker(workout) {
    L.marker(workout.coords)
      .addTo(this.#map)
      .bindPopup(
        L.popup({
          maxWidth: 250,
          minWidth: 100,
          autoClose: false,
          closeOnClick: false,
          className: `${workout.type}-popup`,
        })
      )
      .setPopupContent(
        `${workout.type === 'running' ? '🏃‍♂️' : '🚴‍♀️'} ${workout.discription}`
      )
      .openPopup();
  }

  // Render form values to the list
  _renderWorkout(workout) {
    let html = `
    <li class="workout workout--${workout.type}" data-id="${workout.id}">
          <h2 class="workout__title">${workout.discription}</h2>
          <div class="workout__details">
            <span class="workout__icon">${
              workout.type === 'running' ? '🏃‍♂️' : '🚴‍♀️'
            }</span>
            <span class="workout__value">${workout.distance}</span>
            <span class="workout__unit">km</span>
          </div>
          <div class="workout__details">
            <span class="workout__icon">⏱</span>
            <span class="workout__value">${workout.duration}</span>
            <span class="workout__unit">min</span>
          </div>`;

    if (workout.type === 'running') {
      html += `<div class="workout__details">
      <span class="workout__icon">⚡️</span>
      <span class="workout__value">-${workout.pace.toFixed(1)}</span>
      <span class="workout__unit">min/km</span>
    </div>
    <div class="workout__details">
      <span class="workout__icon">🦶🏼</span>
      <span class="workout__value">-${workout.cadence}</span>
      <span class="workout__unit">spm</span>
    </div>`;
    }

    if (workout.type === 'cycling') {
      html += `<div class="workout__details">
      <span class="workout__icon">⚡️</span>
      <span class="workout__value">${workout.speed.toFixed(1)}</span>
      <span class="workout__unit">km/h</span>
    </div>
    <div class="workout__details">
      <span class="workout__icon">⛰</span>
      <span class="workout__value">${workout.elevationGain}</span>
      <span class="workout__unit">m</span>
    </div>`;
    }
    form.insertAdjacentHTML('afterend', html);
  }

  // Set the clicked workout view to the center of the map
  _moveToPopup(e) {
    const workoutEl = e.target.closest('.workout');
    console.log(workoutEl);
    if (!workoutEl) return;

    const workout = this.#workout.find(
      work => work.id === workoutEl.dataset.id
    );
    this.#map.setView(workout.coords, this.#mapZoomLevel, {
      animate: true,
      pan: {
        duration: 1,
      },
    });
  }

  // Set form data to localstorage for the later retrivel
  _setLocalStorage() {
    localStorage.setItem('workouts', JSON.stringify(this.#workout));
  }

  // Retrieve from localstorage
  _getFromLocalStorage() {
    const data = JSON.parse(localStorage.getItem('workouts'));
    if (data) {
      this.#workout = data;
      // this.#workout.forEach(el => {
      //   this._renderWorkout(el);
      //   this._renderWorkoutMarker(el);
      // });
    }
  }
}

const app = new App();
