<script setup>
import { ref, reactive, watch, computed, onMounted } from "vue";

const COUNT = 5;
const LOGSTASH_ENDPOINT = "http://localhost:8898";
const language = ref("zh-CN");

const languageList = ["zh-CN", "en-US", "fr-FR"];

const addThousandSeperator = (number) => {
  return number.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",");
};

const generateRandomNumber = (min, max) => {
  return Math.floor(Math.random() * (max - min + 1)) + min;
};

const digits = [1, 2, 3, 4, 5, 6, 7, 8, 9, "⟳", 0, "⌫"];

const currentNumber = ref("waiting...");

const stats = reactive({
  startPerformanceMs: 0,
  performanceMs: 0,
  timeForEachElements: [],
  attempts: 0,
});

const clearState = () => {
  stats.timeForEachElements = []
  currentNumber.value = ""
  currentNumberInput.value = ""
  stats.attempts = 0
  record.value = []
};

const startPlay = () => {
  inputRef.value.focus();
  clearState()
  nextNumber()
};

const inputRef = ref(null);

const delayStart = (msec=3000) => {
  clearState()
  testLogstash()

  setTimeout(startPlay, msec)
}

const start = () => {
  speak()
}

const startTiming = () => {
  stats.startPerformanceMs = performance.now();
};

const pronunceSeperator = ref(true);

const logstashStatus = ref("Connecting...")

const testLogstash = () => {
  fetch(LOGSTASH_ENDPOINT)
    .then((res) => {
      if (res.status == 200) {
        logstashStatus.value = "Connected"
      } else {
        logstashStatus.value = "Not connected"
      }
    })
    .catch((err) => {
      logstashStatus.value = "Not connected"
    })
}

onMounted(() => {
  delayStart();
})

const speak = () => {
  var synthesis = window.speechSynthesis;

  // Get the first voice in the list
  var voice = synthesis.getVoices().filter(function (voice) {
    return voice.lang === language.value;
  })[0];

  let number = currentNumber.value;

  if (pronunceSeperator.value) {
    number = addThousandSeperator(number);
  }

  // Create an utterance object
  var utterance = new SpeechSynthesisUtterance(number);

  // Set utterance properties
  utterance.voice = voice;
  utterance.pitch = 1.5;
  utterance.rate = 1.25;
  utterance.volume = 1;

  // Speak the utterance
  synthesis.speak(utterance);
  utterance.onend = startTiming;
};

const getMinMax = (digit) => {
  let min = 0;
  let max = 0;

  if (digit == 0) {
    min = 0;
    max = 9;
  } else {
    min = Math.pow(10, digit - 1);
    max = Math.pow(10, digit) - 1;
  }

  return [min, max];
}

const record = ref([]);

const sendToLogStash = () => {
  fetch(LOGSTASH_ENDPOINT, {
    method: "POST",
    body: JSON.stringify(record.value, (key, value) =>
            typeof value === 'bigint'
                ? value.toString()
                : value // return everything else unchanged
        ),
    headers: { "Content-Type": "application/json" },
  }).then((res) => {
    if (res.status == 200) {
      logstashStatus.value = "Sent"
    } else {
      logstashStatus.value = "Error"
    }
  }).catch((err) => {
    logstashStatus.value = "Unable to send to logstash"
  });
};

const end = () => {
  stats.performanceMs = BigInt(
    (performance.now() - stats.startPerformanceMs).toFixed(0)
  );
  stats.timeForEachElements.push(stats.performanceMs);
  stats.attempts++;

  record.value.push({
    lang: language.value,
    number: currentNumber.value,
    time: stats.performanceMs,
    isCorrect: currentNumberInput.value == currentNumber.value,
    date: new Date().toISOString().slice(0, 10),
    digits: currentNumber.value.length,
  });

  clear();

  if (stats.attempts == COUNT) {
    setTimeout(() => {

      console.log(record.value);
      sendToLogStash();

      alert(
        "Finished, tap ok to restart."
        // 'Avg: ' + (stats.timeForEachElements.reduce((a, b) => a + b, 0n) / BigInt(COUNT)) + 'ms/n' +
        // 'Min: ' + Math.min(...stats.timeForEachElements) + 'ms/n' +
        // 'Max: ' + Math.max(...stats.timeForEachElements) + 'ms/n'
        // 'Median: ' + stats.timeForEachElements.sort()[Math.floor(stats.timeForEachElements.length / 2)] + 'ms'
      );

      delayStart(1000);
    }, 500)
  } else {
    nextNumber();
  }
};

const nextNumber = () => {
  currentNumber.value = generateRandomNumber(...getMinMax(numOfDigits.value)).toString();
  start();
};

const numOfDigits = ref(4);

const currentNumberInput = ref("");

const clear = () => {
  currentNumberInput.value = "";
};

watch(currentNumberInput, (newValue, oldValue) => {
  if (newValue.length === 0) return
  // values were there
  if (newValue.length == currentNumber.value.length) {
    end();
  }
});


const percent = computed(() => {
  return (stats.attempts / COUNT) * 100 + "%";
});
</script>

<template>
  <div
    class="p-4 max-w-2xl mx-auto flex flex-col gap-4 h-screen w-screen lg:p-16 lg:gap-16 "
  >
    <div class="flex gap-4">
      <button
        @click="() => delayStart(2000)"
        class="align-text-top text-white bg-blue-700 hover:bg-blue-800 focus:ring-4 focus:ring-blue-300 font-medium rounded-lg text-2xl px-5 pt-2 pb-2.5 text-center dark:bg-blue-600 dark:hover:bg-blue-700 focus:outline-none dark:focus:ring-blue-800"
      >⟳</button>
      <select
        v-model="language"
        class="bg-neutral-50 border border-neutral-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5 dark:bg-neutral-700 dark:border-neutral-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
      >
        <option v-for="lang in languageList" :value="lang" :key="lang">
          {{ lang }}
        </option>
      </select>

      <select
        class="bg-neutral-50 border border-neutral-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5 dark:bg-neutral-700 dark:border-neutral-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
        v-model="numOfDigits">
        <option v-for="i in 10" :value="i" :key="i">
          {{ i }}
        </option>
        </select>
    </div>
    <div>
    <label class="relative inline-flex items-center cursor-pointer">
      <input type="checkbox" v-model="pronunceSeperator" class="sr-only peer" />
      <div
        class="w-11 h-6 bg-neutral-200 peer-focus:outline-none peer-focus:ring-4 peer-focus:ring-blue-300 dark:peer-focus:ring-blue-800 rounded-full peer dark:bg-neutral-700 peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-neutral-300 after:border after:rounded-full after:h-5 after:w-5 after:transition-all dark:border-neutral-600 peer-checked:bg-blue-600"
      ></div>
      <span class="ml-3 text-sm font-medium text-gray-900 dark:text-gray-300"
        >Pronunce seperator</span
      >
    </label>
    </div>
    <div class="w-full bg-neutral-200 dark:bg-neutral-700 rounded-full h-4">
      <div
        class="bg-blue-500 h-4 rounded-full dark:bg-blue-600"
        :style="`width: ${percent}`"
      ></div>

      <div class="flex gap-3 mt-2 text-sm text-neutral-700 font-medium dark:text-neutral-300">
        <span> {{  logstashStatus }}</span>
        <span> LANG: {{ language }} </span>

        <template v-if="stats.timeForEachElements.length">
          <span> LAST: {{ stats.performanceMs }} ms </span>

          <span>
            AVG:
            {{
              stats.timeForEachElements.reduce((a, b) => a + b, 0n) /
              BigInt(stats.timeForEachElements.length)
            }}
            ms
          </span>
        </template>
      </div>
    </div>


    <!-- <div>{{ currentNumber }}</div> -->

    <div class="text-center py-6 text-lg font-medium">
      <input
        placeholder="Type down the number here"
        v-model="currentNumberInput"
        ref="inputRef"
        autofocus
        class="lg:text-4xl outline-none w-full border-none focus:border-none focus:ring-0 bg-neutral-100 rounded-lg dark:text-white dark:bg-neutral-700"
        tabindex="0"
      />
    </div>

    <div class="grid grid-cols-3 gap-4 max-w-xs mx-auto lg:hidden">
      <template v-for="(digit, index) in digits">
        <button
          class="bg-black/5 dark:bg-white/5 rounded-full h-20 w-20 flex justify-center text-2xl text-blue-500 font-medium items-center dark:text-blue-600"
          :key="index"
          v-if="digit != null"
          @click="
            () => {
              if (digit == '⌫') {
                currentNumberInput = currentNumberInput.slice(0, -1);
              } else if (digit == '⟳') {
                // repronounce
              } else {
                currentNumberInput = currentNumberInput + digit.toString();
              }
            }
          "
        >
          {{ digit }}
        </button>
        <div v-else></div>
      </template>
    </div>

    <div class="grid grid-cols-5 gap-4" :class="numOfDigits > 5 ? 'p-grid-cols-1' : ''">
      <div
        v-for="(res, i) in record"
        :key="i"
        class="text-3xl font-medium font-mono"
        :class="res.isCorrect ? 'num-correct' : 'num-incorrect'"
      >
        {{ res.number }}
      </div>
    </div>

    <div class="stats">
      <div class="text-3xl font-medium">
        <span class="num-correct">{{ stats.attempts }}</span> / {{ COUNT }}
      </div>

    </div>
  </div>
</template>

<style scoped>
.num-correct {
  @apply text-green-500;
}

.num-incorrect {
  @apply text-red-500;
}

.p-grid-cols-1 {
  @apply grid-cols-1;
}

</style>
