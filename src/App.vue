<script>
import axios from "axios";
import Chart from "chart.js/auto";
import moment from "moment";

export default {
  data() {
    return {
      isToggled: null,
      today: null,
      weatherData: null,
      forecastData: [],
      recentCities: [],
      selectedDay: null,
      showChart: false,
      city: "",
      day: "",
      unit: "°C",
      apiKey: "1ca7258aa3d87284c748defdf36706b5",
      selectedDayForecast: null,
      selectedHourlyForecast: null,
      hourlyChart: null,
      weeklyForecasts: [],
      weatherIcon: "",
      temperature: null,
      description: null,
      iconUrl: null,
      date: null,
      time: null,
      name: null,
      country: null,
      forecast: [],
      dayOfWeek: "",
    };
  },
  mounted() {
    this.loadRecentCities();
    if (this.recentCities.length)
      this.getWeatherForRecentCity(this.recentCities[0].id);
  },
  computed: {
    sysCountry() {
      return this.weatherData?.sys?.country ?? "";
    },
  },
  methods: {
    searchWeather() {
      axios
        .get(
          `https://api.openweathermap.org/data/2.5/weather?q=${this.city}&units=metric&appid=${this.apiKey}`
        )
        .then((response) => {
          this.weatherData = response.data;

          this.addToRecentCities(response.data);
          const d = new Date();
          this.today = this.getDayOfWeek(d);
          this.date =
            d.getDate() +
            "-" +
            (this.monthNames?.length ? this.monthNames[d.getMonth()] : "") +
            "-" +
            d.getFullYear();
          this.time =
            d.getHours() + ":" + d.getMinutes() + ":" + d.getSeconds();
        })
        .catch((error) => {
          console.error(error);
        });

      axios
        .get(
          `https://api.openweathermap.org/data/2.5/forecast?q=${this.city}&appid=${this.apiKey}&units=metric`
        )
        .then((response) => {
          this.forecastData = response.data.list
            .filter((day) => day.dt_txt.includes("12:00:00"))
            .slice(0, 5)
            .map((day) => ({
              date: day.dt_txt,
              icon: day.weather[0].icon,
              temperature: day.main.temp,
              wind: day.wind,
              main: day.main,
              day: new Date(day.dt_txt).toLocaleDateString("en-US", {
                weekday: "short",
              }),
            }));
        })

        .catch((error) => {
          console.error(error);
        });
    },
    getWeatherForRecentCity(cityId) {
      const recentCity = this.recentCities.find((city) => city.id === cityId);
      this.weatherData = recentCity;
      this.city = recentCity.name;
      this.searchWeather();
      this.selectDayForecast();
    },
    addToRecentCities(cityData) {
      if (!this.recentCities.some((city) => city.id === cityData.id)) {
        this.recentCities.unshift(cityData);
        if (this.recentCities.length > 3) {
          this.recentCities.pop();
        }
        localStorage.setItem("recentCities", JSON.stringify(this.recentCities));
      }
    },
    saveRecentCity(city, country, temperature) {
      let recentCities = JSON.parse(localStorage.getItem("recentCities")) || [];
      recentCities.unshift({ city, country, temperature });
      recentCities = recentCities.slice(0, 3);
      localStorage.setItem("recentCities", JSON.stringify(recentCities));
      this.recentCities = recentCities;
    },
    loadRecentCities() {
      const recentCities = JSON.parse(localStorage.getItem("recentCities"));
      if (recentCities) {
        this.recentCities = recentCities;
      }
    },
    convertTemperature(temp) {
      if (!temp) return "";
      if (this.unit === "°C") {
        return temp;
      } else {
        return (temp * 9) / 5 + 32;
      }
    },
    toggleUnit() {
      if (this.unit === "°C") {
        this.unit = "°F";
      } else {
        this.unit = "°C";
      }
    },
    getWeatherIconURL(iconCode) {
      return `https://api.openweathermap.org/img/w/${iconCode}.png`;
    },
    convertUnixTime(unixTime) {
      return new Date(unixTime * 1000);
    },
    selectDayForecast(day) {
      this.selectedDayForecast = day;
    },
    async selectDayForecast(day) {
      this.selectedDayForecast = day;
      const response = await axios.get(
        `https://api.openweathermap.org/data/2.5/forecast?q=${this.city}&appid=${this.apiKey}&units=metric`
      );
      const forecastData = response.data.list;
      this.selectedDayTemperature = day?.main?.temp ?? forecastData[0];
      this.selectedDayForecast = day?.main?.temp ?? forecastData[0];

      const selectedDayData = forecastData.filter((data) => {
        return (
          new Date(data.dt * 1000).toDateString() ===
          new Date((day?.dt ?? forecastData[0]) * 1000).toDateString()
        );
      });
      const labels = selectedDayData.map(
        (data) => new Date(data.dt * 1000).getHours() + "AM"
      );
      const temperatures = selectedDayData.map((data) => data.main.temp);
      await this.$nextTick();

      const ctx = document.getElementById("hourlyChart");
      if (this.hourlyChart) {
        this.hourlyChart.destroy();
      }
      this.hourlyChart = new Chart(ctx, {
        type: "bar",
        data: {
          labels: labels,
          datasets: [
            {
              label: "Hourly Temperature (°C)",
              data: temperatures,
              backgroundColor: "rgba(75, 192, 192, 0.2)",
              borderColor: "rgba(75, 192, 192, 1)",
              borderWidth: 1,
            },
          ],
        },

        options: {
          scales: {
            y: {
              beginAtZero: true,
              title: {
                display: true,
                text: "temprature(c)",
              },
            },
            x: {
              title: {
                display: true,
                text: "time",
              },
            },
          },
        },
      });
    },
    SunTime(timeSun) {
      const date = new Date(timeSun * 1000);
      return date.toLocaleTimeString([], {
        hour: "2-digit",
        minute: "2-digit",
        hour12: false,
      });
    },
    getDayOfWeek(date) {
      const daysOfWeek = [
        "sunday",
        "monday",
        "tuesday",
        "wednesday",
        "thursday",
        "friday",
        "saturday",
      ];
      const d = new Date(date * 1000);
      return daysOfWeek[d.getDay()];
    },
  },
  filters: {
    formatDate(date) {
      return date.toLocaleDateString("fa-IR");
    },
  },
};
</script>
<template>
  <div class="contener">
    <div class="flex flex-row justify-between px-[24px] pt-4 pb-[40px]">
      <div class="flex flex-row justify-between gap-12">
        <div class="flex gap-3 pr-4" v-if="weatherData">
          <div>
            <img src="../images/location.svg" alt="location" class="pt-2" />
          </div>
          <div
            class="flex pt-2 font-Montserrat font-normal text-[16px] text-[#FEFEFE]"
          >
            <p>{{ weatherData.name }}</p>
            ,
            <p>{{ sysCountry }}</p>
          </div>
        </div>

        <div
          class="flex flex-row gap-2 bg-[#1E1E1E] rounded-[10px] px-4 py-2 w-[500px]"
        >
          <img src="../images/search.svg" alt="" class="" />
          <input
            type="text"
            v-model="city"
            @keyup.enter="searchWeather"
            class="text-[#EDEDED] bg-[#1E1E1E] font-normal text-sm w-[500px]"
            placeholder="Search City"
          />
        </div>
      </div>

      <div>
        <input
          type="checkbox"
          id="toggle"
          v-model="isToggled"
          class="toggle-checkbox"
          @click="toggleUnit"
        />
        <label for="toggle" class="toggle-label">
          <div class="toggle-inner"></div>
          <div class="toggle-slider"><span class="p-[20%]"> &deg;c</span></div>
        </label>
      </div>
    </div>
    <div class="flex flex-row justify-between px-[24px] h-96 pt-4 gap-20">
      <div class="flex flex-col gap-[16px]">
        <h2
          class="font-Montserrat font-medium text-[20px] leading-[24.3px] text-[#FFFFFF]"
        >
          Today
        </h2>
        <div class="flex flex-col h-[226px]">
          <div>
            <div v-if="weatherData">
              <div
                class="flex flex-row justify-between items-center bg-[#AECADF] rounded-t-3xl h-[52px] p-[16px]"
              >
                <span
                  class="font-Montserrat font-semibold text-[16px] leading-[19.5px] text-[#0F0F11]"
                >
                  {{ getDayOfWeek(weatherData.dt) }}</span
                >
                <span
                  class="font-Montserrat font-semibold text-[16px] leading-[19.5px] text-[#0F0F11]"
                  >{{ time }} AM</span
                >
              </div>
              <div class="bg-[#BBD7EC] w-[257px] rounded-b-3xl">
                <div class="flex flex-row gap-[2px] justify-around">
                  <span
                    class="font-semibold font-Montserrat p-[16px] text-[38px] leading-[78.02px]"
                  >
                    {{ convertTemperature(Math.round(weatherData.main.temp)) }}
                    &deg;</span
                  >
                  <img
                    :src="getWeatherIconURL(weatherData.weather[0].icon)"
                    :alt="weatherData.weather[0].description"
                  />
                </div>
                <div class="p-[16px] grid grid-cols-2">
                  <div
                    class="text-[#4F5658] flex w-40 gap-2 font-Montserrat leading-[18px] text-[14px] font-normal"
                  >
                    Feels like<span
                      class="text-[#0F0F11] font-Montserrat leading-[18px] font-semibold text-[12px] flex"
                    >
                      {{
                        convertTemperature(
                          Math.round(weatherData.main.feels_like)
                        )
                      }}
                      &deg;
                    </span>
                  </div>
                  <div
                    class="text-[#4F5658] flex w-40 gap-2 font-Montserrat leading-[18px] text-[12px] font-normal"
                  >
                    Sunrise
                    <span
                      class="text-[#0F0F11] font-Montserrat leading-[18px] font-semibold text-[12px]"
                    >
                      {{ SunTime(weatherData.sys.sunrise) }}
                    </span>
                  </div>
                  <div
                    class="text-[#214c58] flex w-40 gap-2 font-Montserrat leading-[18px] text-[12px] font-normal"
                  >
                    Pressure
                    <span
                      class="text-[#0F0F11] font-Montserrat leading-[18px] font-semibold text-[12px]"
                    >
                      {{ weatherData.main.pressure }}mb</span
                    >
                  </div>
                  <div
                    class="text-[#4F5658] flex w-40 gap-2 font-Montserrat leading-[18px] text-[12px] font-normal"
                  >
                    Sunset
                    <span
                      class="text-[#0F0F11] font-Montserrat leading-[18px] font-semibold text-[12px]"
                    >
                      {{ SunTime(weatherData.sys.sunset) }}
                    </span>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="flex flex-col gap-[16px]">
        <h2
          class="font-Montserrat font-medium text-[15px] leading-[24px] text-[#fff]"
        >
          Next 5 Days
        </h2>
        <div
          class="rounded-[30px] flex flex-col justify-between items-center bg-[#1B1B1D] h-[226px] w-[84px]"
        >
          <ul
            v-if="forecastData"
            class="flex flex-row gap-[16px] w-[580px] ml-[490px]"
          >
            <li
              v-for="day in forecastData"
              :key="day.dt"
              @click="selectDayForecast(day)"
              class="rounded-[30px] flex flex-col justify-between items-center bg-[#1f1f22] h-[226px] w-[84px]"
            >
              <div class="px-[26px] py-[12px]">
                <div
                  class="font-Montserrat font-semibold text-[16px] leading-[19px] text-[rgb(229,229,229)] border-b border-[#39393A] px-[26px] py-[12px]"
                >
                  {{ day.day }}
                </div>
                <img
                  :src="`http://openweathermap.org/img/w/${day.icon}.png`"
                  alt="icon"
                  class="pl-8 pt-4"
                />
              </div>
              <div
                class="box-border pb-[30px] text-[#FDFDFD] font-Montserrat font-semibold text-[18px] leading-[39px] px-[18px] py-[12px]"
              >
                {{ convertTemperature(Math.round(day.temperature)) }} &deg;
              </div>
            </li>
          </ul>
        </div>
      </div>
      <div v-if="selectedDayForecast" class="flex flex-col gap-[16px] pl-96">
        <h2>
          <span class="font-Montserrat font-medium text-xl text-[#FFF]"
            >Monday</span
          ><span class="font-Montserrat font-medium text-xl text-[#626161]"
            ><div>
              <h3>Hourly Temperature Forecast</h3>
              <canvas id="hourlyChart" width="400" height="200"></canvas></div
          ></span>
        </h2>
        <div class="w-[409px] h-[226px]"></div>
      </div>
    </div>
    <div class="flex">
      <div
        v-if="selectedDayForecast"
        class="pt-[10px] flex flex-row px-[24px] pb-4"
      >
        <div class="flex flex-col gap-4">
          <div class="flex">
            <div class="h-6 font-medium text-[20px] leading-6 text-white">
              Monday
            </div>

            <div
              class="w-[198px] h-6 font-medium text-[20px] leading-6 text-[#626161]"
            >
              ’s Overview
            </div>
          </div>
          <div class="grid grid-cols-2 gap-[16px]">
            <div
              class="flex flex-col justify-between rounded-2xl p-3 w-[180px] h-[180px] border border-black bg-[#1f1f22]"
            >
              <div
                class="font-normal font-Montserrat text-[16px] text-[#fff] leading-5"
              >
                High Temp
              </div>
              <div class="flex justify-center">
                <img
                  class="h-[48px] w-[48px]"
                  src="../images/high.svg"
                  alt="high temp"
                />
              </div>
              <div
                class="font-Montserrat flex justify-center font-semibold text-xl text-[#fff]"
              >
                {{
                  selectedDayForecast?.main?.temp_max
                    ? convertTemperature(
                        Math.round(selectedDayForecast.main.temp_max)
                      )
                    : ""
                }}{{ unit }}
              </div>
            </div>
            <div
              class="flex flex-col justify-between rounded-2xl p-3 w-[180px] h-[180px] border border-black bg-[#1B1B1D]"
            >
              <div
                class="font-normal font-Montserrat text-[16px] text-[#fff] leading-5"
              >
                low Temp
              </div>
              <div class="flex justify-center">
                <img
                  class="h-[48px] w-[48px]"
                  src="../images/low.svg"
                  alt="low temp"
                />
              </div>
              <div
                class="font-Montserrat flex justify-center font-semibold text-xl text-[#fff]"
              >
                {{
                  convertTemperature(
                    Math.round(selectedDayForecast?.main?.temp_min)
                  )
                }}{{ unit }}
              </div>
            </div>
            <div
              class="flex flex-col justify-between rounded-2xl p-3 w-[180px] h-[180px] border border-black bg-[#1B1B1D]"
            >
              <div
                class="font-normal font-Montserrat text-[16px] text-[#fff] leading-5"
              >
                wind
              </div>
              <div class="flex justify-center">
                <img
                  class="h-[48px] w-[48px]"
                  src="../images/wind.svg"
                  alt="wind"
                />
              </div>
              <div
                class="font-Montserrat flex justify-center font-semibold text-xl text-[#fff]"
              >
                {{ selectedDayForecast.wind.speed }} m/s
              </div>
              <div class="text-[#fff]">
                جهت باد: {{ selectedDayForecast.wind.deg }}
              </div>
            </div>
            <div
              class="flex flex-col justify-between rounded-2xl p-3 w-[180px] h-[180px] border border-black bg-[#1B1B1D]"
            >
              <div
                class="font-normal font-Montserrat text-[16px] text-[#fff] leading-5"
              >
                humidity
              </div>
              <div class="flex justify-center">
                <img
                  class="h-[48px] w-[48px]"
                  src="../images/humidity.svg"
                  alt="humidity"
                />
              </div>
              <div
                class="font-Montserrat flex justify-center font-semibold text-xl text-[#fff]"
              >
                {{ selectedDayForecast.main.humidity }}%
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="w-[526px] h-[414px] rounded-2xl">
        <div
          class="w-[526px] h-[436px] rounded-2xl bg-[url('../images/background.svg')] bg-cover flex justify-center items-center"
        >
          <div
            class="w-[360px] h-[80px] font-Montserrat font-medium text-xl leanding-[32px] text-[#fff] border border-[#fff] rounded-lg px-4 py-2 backdrop-filter backdrop-blur-sm"
          >
            Explore Global Map Of Wind Weather And Ocean Conditions
          </div>
        </div>
      </div>
      <div class="flex flex-col gap-4 w-[526] h-[414px]">
        <h2 class="font-Montserrat font-medium text-xl text-[#fff] pl-5">
          Recent Cities
        </h2>
        <ul>
          <li
            class="flex flex-col gap-4 pb-3 pl-5 h-32"
            v-for="recentCity in recentCities"
            :key="recentCity.id"
            @click="getWeatherForRecentCity(recentCity.id)"
          >
            <div
              class="flex flex-row justify-between rounded-2xl w-[297px] px-6 py-3 bg-[#1f1f22]"
            >
              <div class="flex flex-col w-[297px]">
                <p class="font-Montserrat font-normal text-[#777] text-sm">
                  {{ recentCity.sys.country }}
                </p>
                <p class="font-Montserrat font-normal text-xl text-[#fff]">
                  {{ recentCity.name }}
                </p>
                <p class="font-Montserrat font-normal text-sm text-[#EFEFEF]">
                  {{ recentCity.weather[0].description }}
                </p>
              </div>
              <img
                :src="getWeatherIconURL(recentCity.weather[0].icon)"
                :alt="recentCity.weather[0].description"
              />
            </div>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<style scoped>
.contener {
  background-color: #19191b;
  margin: 0;
  padding: 0;
}

span {
  display: flex;
  flex-direction: row;
}

.toggle-checkbox {
  display: none;
}

.toggle-label {
  position: relative;
}

.toggle-inner {
  width: 80px;
  height: 40px;
  border: 1px solid #626161;
  border-radius: 24px;
  transition: background-color 0.3s;
}

.toggle-slider {
  position: absolute;
  top: 9%;
  left: 0;
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background-color: #d8e9f9;
  transition: left 0.3s;
}

input[type="checkbox"]:checked + .toggle-label .toggle-inner {
  background-color: #19191b; /* Color when toggle is on */
}

input[type="checkbox"]:checked + .toggle-label .toggle-slider {
  left: 50px;
}
</style>





































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































