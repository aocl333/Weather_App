<template>
    <div class="rightContainer">
        <div id="cityNameBox">
            <div class="cityName">
                <p>{{ cityName }}</p>
                <p>{{ currentTime }}</p>
            </div>
        </div>
        <div id="contentsBox">
            <div class="buttonBox">
                <div class="buttonBackground">
                    <button class="forecast">Forecast</button>
                    <button class="airquality">Air Quality</button>
                </div>
            </div>
            <div class="weatherBox">
                <div class="airCondition">
                    <p>{{ feeling }}</p>
                </div>
                <div class="detail">
                    <div class="title">
                        <p>πμμΈν λ μ¨ λ°μ΄ν°π</p>
                    </div>
                    <div class="data" v-for="(detailData, index) in subWeatherData" :key="index">
                        <div class="dataName">
                            <p></p>
                            <p>{{ detailData.name }}</p>
                        </div>
                        <div class="dataValue">
                            <p>{{ detailData.value }}<span></span></p>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <Map />
        <nav>
            <i class="fas fa-home"></i>
            <i class="fas fa-search-location"></i>
            <i class="fas fa-chart-line"></i>
            <i class="fas fa-cog"></i>
        </nav>
    </div>
</template>

<script>
import Map from '~/components/Map.vue';
import { computed, onMounted, ref, watchEffect } from 'vue';
import { useStore } from 'vuex';
import dayjs from 'dayjs';
import 'dayjs/locale/ko';
dayjs.locale('ko'); // globalλ‘ νκ΅­μ΄ locale μ¬μ©

export default {
    components: {
        Map,
    },

    setup() {
        // νλ©΄μμ λ³΄μ¬μ§ λ°μ΄ν°
        let currentTime = dayjs().format('YYYY. MM. DD. ddd'); // νμ¬ μκ°

        let cityName = ref(''); // λμ μ΄λ¦
        let feeling = ref(''); // νμ¬ μ¨λμ λν μ²΄κ°μ λνλ΄λ λ°μ΄ν°
        let subWeatherData = ref([]); // μμΈ λ μ¨ λ°μ΄ν°

        // νμμ€ν¬νλ‘ λ³ν
        const Unix_timestamp = (dt) => {
            let date = new Date(dt * 1000);
            // padStart() λ©μλλ νμ¬ λ¬Έμμ΄μ μμμ λ€λ₯Έ λ¬Έμμ΄λ‘ μ±μ, μ£Όμ΄μ§ κΈΈμ΄λ₯Ό λ§μ‘±νλ μλ‘μ΄ λ¬Έμμ΄μ λ°νν©λλ€.
            // μ±μλ£κΈ°λ λμ λ¬Έμμ΄μ μμ(μ’μΈ‘)λΆν° μ μ©λ©λλ€.
            let hour = date.getHours().toString().padStart(2, '0');
            return hour.substring(-2) + 'μ';
        };

        // OpenWeatherAPI νΈμΆ ν¨μ
        const store = useStore();
        const fetchOpenWeatherApi = async () => {
            // API νΈμΆμ μν νμ λ°μ΄ν°
            try {
                await store.dispatch('openWeatherApi/FETCH_OPENWEATHER_API');
                const { currentFeelsLike, currentSunrise, currentSunset, currentVisibility } = store.state.openWeatherApi.currentWeather;

                let isInitialCityName = store.state.openWeatherApi.cityName; // μ΄κΈ° λμμ΄λ¦ λ°μ΄ν°
                let isFeelLikeTemp = computed(() => {
                    return currentFeelsLike;
                }); // μ΄κΈ° μ²΄κ°μ¨λ λ°μ΄ν°
                let isTimeOfSunrise = computed(() => {
                    return currentSunrise;
                }); // μΌμΆμκ° λ°μ΄ν°
                let isTimeOfSunSet = computed(() => {
                    return currentSunset;
                }); // μΌλͺ°μκ° λ°μ΄ν°
                let isLineOfSight = computed(() => {
                    return currentVisibility;
                }); // κ°μκ±°λ¦¬ λ°μ΄ν°

                // κΈ°μ€μ μ λ°λ₯Έ Arrayλ₯Ό νλ λ§λ€κ³ 
                // κΈ°μ€μ λ°λ₯Έ λ©μμ§μ λ°λ₯Έ Arrayλ₯Ό νλ λ§λ€μ΄μ
                // μ²΄κ°μ¨λ λ°μ΄ν°κ° νμμ ν΄μ
                // μνλ κ°μ λ½λ λ‘μ§μΌλ‘ κ΅¬μ±

                const pivots = [0, 10, 15, 20, 25, 30];
                const labels = ['λ§€μ° μΆμ', 'μΆμ', 'μμν¨', 'μ μ ν¨', 'λ³΄ν΅', 'λμ', 'λ§€μ° λμ'];

                let index = 0;
                for (const p of pivots) {
                    if (isFeelLikeTemp.value <= p) break;
                    index++;
                }
                feeling.value = labels[index];

                // κ°κ³΅ν λ°μ΄ν°λ₯Ό κ°μ§κ³  μλ‘μ΄ λ°°μ΄ μμ±
                let isProcessedData = [
                    { name: 'μΌμΆμκ°', value: Unix_timestamp(isTimeOfSunrise.value) },
                    {
                        name: 'μΌλͺ°μκ°',
                        value: Unix_timestamp(isTimeOfSunSet.value),
                    },
                    {
                        name: 'κ°μκ±°λ¦¬',
                        value: isLineOfSight.value.toString().replace(/\B(?<!\.\d*)(?=(\d{3})+(?!\d))/g, ',') + 'M',
                    },
                ];

                // Composition APIμμ AJAXμμ²­κ³Ό λ°μ΄ν° λ³κ²½μ νλ €λ©΄ λ°μ΄ν°.valueλ‘ μ κ·Όν΄μΌνλ€.
                cityName.value = isInitialCityName;
                subWeatherData.value = isProcessedData;
            } catch (error) {
                console.log(error);
                // alert('APIκ° μ λλ‘ νΈμΆλμ§ μμμ΅λλ€.');
            }
        };

        watchEffect(async () => {
            await fetchOpenWeatherApi();
        });

        onMounted(() => {
            fetchOpenWeatherApi();
        });

        return {
            cityName,
            currentTime,
            feeling,
            subWeatherData,
        };
    },
};
</script>

<style lang="scss" scoped>
@import '~/scss/main.scss';
@import '~/scss/subview.scss';
</style>
