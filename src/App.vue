<template>
  <div class="app">
    <video autoplay muted loop id="background-video">
      <source src="@/assets/background.mp4" type="video/mp4" />
    </video>
    
    <div class="content">
      <h1>Password Impossible</h1>
      <div class="password-input">
        <input
          type="text"
          v-model="password"
          placeholder="Type your password"
        />
        <div class="strength-meter" :style="{ width: `${(passedRules.length / rules.length) * 100}%` }"></div>
      </div>
      <div class="rules-container">
        <transition-group name="rule">
          <div
            v-for="rule in visibleRules"
            :key="rule.id"
            :class="['rule', { passed: passedRules.includes(rule.id) }]"
          >
            <span class="rule-icon">{{ passedRules.includes(rule.id) ? '✓' : '!' }}</span>
            <span class="rule-text">{{ rule.description }}</span>
          </div>
        </transition-group>
      </div>
      
      <div v-if="showCenaChallenge" class="cena-challenge">
        <video width="320" height="240" autoplay @ended="startMovingButton" ref="cenaVideo">
          <source src="@/assets/cena.mp4" type="video/mp4">
        </video>
        <button 
          v-if="showMovingButton"
          @click="catchButton" 
          :style="{ left: cenaButtonPosition.x + 'px', top: cenaButtonPosition.y + 'px' }"
          class="moving-button"
        >
          YES
        </button>
      </div>
      
      <transition name="fade">
        <div v-if="passedRules.length === rules.length && !showJumpScare && !showCongratulations" class="success-message">
          Congratulations! You've created an impossible password!
        </div>
      </transition>
    </div>

    <div v-if="showJumpScare" class="jump-scare">
      <video autoplay @ended="endJumpScare" ref="jumpScareVideo">
        <source src="@/assets/jumpscare.mp4" type="video/mp4">
      </video>
    </div>

    <div v-if="showCongratulations" class="celebration-background"></div>

    <transition name="fade">
      <div v-if="showCongratulations" class="congratulations">
        <h2>🎉 Congratulations! 🎉</h2>
        <p>You've conquered Password Impossible!</p>
        
        <div class="password-display">
          <div class="password-label">YOUR IMPOSSIBLE PASSWORD:</div>
          <div class="password-value">{{ password }}</div>
        </div>
        
        <div class="time-wasted">
          ON WASTING {{ elapsedTime }} OF YOUR LIFE
        </div>
      </div>
    </transition>
  </div>
</template>

<script>
import { ref, watch, onMounted, computed } from 'vue'
import confetti from 'canvas-confetti'

function getTodayDate() {
  const today = new Date();
  return String(today.getDate()).padStart(2, '0');
}

function formatTime(milliseconds) {
  const seconds = Math.floor(milliseconds / 1000);
  const minutes = Math.floor(seconds / 60);
  const hours = Math.floor(minutes / 60);

  if (hours > 0) {
    return `${hours} HOUR${hours !== 1 ? 'S' : ''} AND ${minutes % 60} MINUTE${minutes % 60 !== 1 ? 'S' : ''}`;
  } else if (minutes > 0) {
    return `${minutes} MINUTE${minutes !== 1 ? 'S' : ''} AND ${seconds % 60} SECOND${seconds % 60 !== 1 ? 'S' : ''}`;
  } else {
    return `${seconds} SECOND${seconds !== 1 ? 'S' : ''}`;
  }
}

const rules = [
  { id: 1, description: "Your password must be at least 8 characters", check: (pwd) => pwd.length >= 8 },
  { id: 2, description: "Your password must include a number", check: (pwd) => /\d/.test(pwd) },
  { id: 3, description: "Your password must include an uppercase letter", check: (pwd) => /[A-Z]/.test(pwd) },
  { id: 4, description: "Your password must include a special character", check: (pwd) => /[!@#$%^&*(),.?":{}|<>]/.test(pwd) },
  { 
    id: 5, 
    description: "Your password must include a month of the year", 
    check: (pwd) => {
      const months = ["january", "february", "march", "april", "may", "june", "july", "august", "september", "october", "november", "december"];
      const lowercasePwd = pwd.toLowerCase();
      return months.some(month => lowercasePwd.includes(month));
    }
  },
  { 
    id: 6, 
    description: `Your password must include today's date`, 
    check: (pwd) => {
      const date = getTodayDate();
      return pwd.includes(date) || pwd.includes(String(parseInt(date, 10)));
    }
  },
  {
    id: 7,
    description: "Your password must include your current T-Shirt Size [Small, Medium, Large]",
    check: (pwd) => {
      const sizes = ["small", "medium", "large"];
      const lowercasePwd = pwd.toLowerCase();
      return sizes.some(size => lowercasePwd.includes(size));
    }
  },
  {
    id: 8,
    description: "Are you sure about that?",
    check: () => false,
  },
  {
    id: 9,
    description: "Your password must include a continent name",
    check: (pwd) => {
      const continents = ["africa", "antarctica", "asia", "australia", "europe", "north america", "south america"];
      const lowercasePwd = pwd.toLowerCase();
      return continents.some(continent => lowercasePwd.includes(continent));
    }
  },
]

export default {
  name: 'App',
  setup() {
    const password = ref('')
    const activeRules = ref([])
    const passedRules = ref([])
    const showCenaChallenge = ref(false)
    const showMovingButton = ref(false)
    const cenaButtonPosition = ref({ x: 0, y: 0 })
    const challengeCompleted = ref(false)
    const showJumpScare = ref(false)
    const gameCompleted = ref(false)
    const showCongratulations = ref(false)
    const startTime = ref(Date.now())
    const endTime = ref(null)
    const elapsedTime = ref('')
    let moveInterval

    const visibleRules = computed(() => {
      return activeRules.value.slice(-5)
    })

    function fireCelebrationConfetti() {
      const duration = 5000;
      const animationEnd = Date.now() + duration;
      const defaults = { startVelocity: 30, spread: 360, ticks: 60, zIndex: 1500 };

      function randomInRange(min, max) {
        return Math.random() * (max - min) + min;
      }

      const interval = setInterval(function() {
        const timeLeft = animationEnd - Date.now();

        if (timeLeft <= 0) {
          return clearInterval(interval);
        }

        const particleCount = 50 * (timeLeft / duration);
        
        confetti({
          ...defaults,
          particleCount,
          origin: { x: randomInRange(0.1, 0.3), y: Math.random() - 0.2 }
        });
        confetti({
          ...defaults,
          particleCount,
          origin: { x: randomInRange(0.7, 0.9), y: Math.random() - 0.2 }
        });
      }, 250);
    }

    const checkRules = () => {
      let newPassedRules = []
      let newActiveRules = []

      for (let rule of rules) {
        if (rule.id === 8 && challengeCompleted.value) {
          newPassedRules.push(rule.id)
          newActiveRules.push(rule)
        } else if (rule.check(password.value)) {
          newPassedRules.push(rule.id)
          newActiveRules.push(rule)
        } else {
          newActiveRules.push(rule)
          break
        }
      }

      passedRules.value = newPassedRules
      activeRules.value = newActiveRules

      if (activeRules.value.some(rule => rule.id === 8) && !showCenaChallenge.value && !challengeCompleted.value) {
        showCenaChallenge.value = true
      }

      if (newPassedRules.length === rules.length && !gameCompleted.value) {
        gameCompleted.value = true
        endTime.value = Date.now()
        elapsedTime.value = formatTime(endTime.value - startTime.value)
        triggerJumpScare()
      }
    }

    function startMovingButton() {
      showMovingButton.value = true
      moveButton()
      setTimeout(() => {
        clearInterval(moveInterval)
        cenaButtonPosition.value = {
          x: window.innerWidth / 2 - 50,
          y: window.innerHeight / 2 - 25
        }
      }, 10000)
    }

    function moveButton() {
      moveInterval = setInterval(() => {
        cenaButtonPosition.value = {
          x: Math.random() * (window.innerWidth - 100),
          y: Math.random() * (window.innerHeight - 50)
        }
      }, 500)
    }

    function catchButton() {
      challengeCompleted.value = true
      showCenaChallenge.value = false
      showMovingButton.value = false
      clearInterval(moveInterval)
      checkRules()
    }

    function triggerJumpScare() {
      showJumpScare.value = true
    }

    function endJumpScare() {
      showJumpScare.value = false
      showCongratulations.value = true
      fireCelebrationConfetti()
    }

    watch(password, checkRules)

    onMounted(() => {
      startTime.value = Date.now()
      checkRules()
    })

    return {
      password,
      visibleRules,
      passedRules,
      rules,
      showCenaChallenge,
      showMovingButton,
      cenaButtonPosition,
      challengeCompleted,
      startMovingButton,
      catchButton,
      showJumpScare,
      endJumpScare,
      showCongratulations,
      elapsedTime,
    }
  }
}
</script>

<style>
@import url('https://fonts.cdnfonts.com/css/jmh-typewriter');
@import url('https://fonts.googleapis.com/css2?family=Kalnia&display=swap');

body, html {
  margin: 0;
  padding: 0;
  height: 100%;
  overflow: hidden;
}

.app {
  font-family: 'Kalnia', serif;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  color: #333;
  position: relative;
  overflow: hidden;
}

#background-video {
  position: fixed;
  right: 0;
  bottom: 0;
  min-width: 100%;
  min-height: 100%;
  width: auto;
  height: auto;
  z-index: -2;
  object-fit: cover;
}

.content {
  background-color: rgba(223, 232, 251, 0.8);
  padding: 2.5rem;
  border-radius: 20px;
  max-width: 500px;
  width: 90%;
  box-sizing: border-box;
  box-shadow: 0 15px 30px rgba(0, 0, 0, 0.1), 0 5px 15px rgba(0, 0, 0, 0.07);
  backdrop-filter: blur(10px);
  transition: all 0.3s ease;
}

.content:hover {
  transform: translateY(-5px);
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.15), 0 10px 20px rgba(0, 0, 0, 0.1);
}

h1 {
  font-family: 'JMH Typewriter', sans-serif;
  font-size: 2.2rem;
  margin-bottom: 1.5rem;
  text-align: center;
  color: #2c3e50;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.1);
}

.password-input {
  position: relative;
  margin-bottom: 1.5rem;
  width: 100%;
  display: flex;
  justify-content: center;
}

input {
  width: 100%;
  max-width: 400px;
  padding: 15px;
  font-size: 16px;
  background-color: #fff;
  border: 2px solid #ddd;
  border-radius: 12px;
  color: #333;
  font-family: 'JMH Typewriter', sans-serif;
  transition: all 0.3s ease;
  box-shadow: 0 2px 5px rgba(0,0,0,0.1);
}

input:focus {
  outline: none;
  border-color: #3498db;
  box-shadow: 0 0 0 3px rgba(52, 152, 219, 0.3);
}

.strength-meter {
  position: absolute;
  bottom: -2px;
  left: 50%;
  transform: translateX(-50%);
  height: 4px;
  background-color: #3498db;
  transition: width 0.5s ease;
  border-radius: 0 0 12px 12px;
  max-width: 400px;
  width: calc(100% - 30px);
}

.rules-container {
  background-color: #f8f9fa;
  border-radius: 12px;
  padding: 1.2rem;
  margin-top: 1.5rem;
  box-shadow: inset 0 2px 4px rgba(0,0,0,0.06);
}

.rule {
  display: flex;
  align-items: flex-start;
  margin-bottom: 0.8rem;
  padding: 0.8rem;
  border-radius: 8px;
  transition: all 0.3s ease;
  background-color: #fefccd;
  box-shadow: 0 2px 4px rgba(0,0,0,0.04);
}

.rule-icon {
  flex-shrink: 0;
  display: inline-flex;
  justify-content: center;
  align-items: center;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  margin-right: 12px;
  font-size: 14px;
  color: white;
  background-color: #e74c3c;
  transition: all 0.3s ease;
}

.rule.passed {
  background-color: #e8f5e9;
}

.rule.passed .rule-icon {
  background-color: #2ecc71;
}

.rule-text {
  font-size: 0.95rem;
  line-height: 1.2;
  flex-grow: 1;
  display: flex;
  align-items: center;
  min-height: 28px;
}

.success-message {
  margin-top: 1.5rem;
  padding: 1rem;
  background-color: #d4edda;
  border: 1px solid #c3e6cb;
  border-radius: 12px;
  color: #155724;
  font-weight: bold;
  text-align: center;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  transition: all 0.3s ease;
}

.rule-enter-active, .rule-leave-active {
  transition: all 0.5s ease;
}
.rule-enter-from, .rule-leave-to {
  opacity: 0;
  transform: translateX(-20px);
}

.fade-enter-active, .fade-leave-active {
  transition: opacity 0.5s ease;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}

.cena-challenge {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.8);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.cena-challenge video {
  max-width: 80%;
  max-height: 80%;
}

.moving-button {
  position: fixed;
  padding: 10px 20px;
  background-color: #e74c3c;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: all 0.1s ease;
  z-index: 1001;
  width: 100px;
  height: 50px;
}

.moving-button:hover {
  transform: scale(1.1);
}

.jump-scare {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 2000;
  background-color: black;
  display: flex;
  justify-content: center;
  align-items: center;
}

.jump-scare video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.congratulations {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 1500;
  background-color: rgba(255, 255, 255, 0.95);
  padding: 3rem 4rem;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 15px 40px rgba(0, 0, 0, 0.3);
  animation: popIn 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275) forwards;
  max-width: 90%;
  width: 600px;
}

.congratulations h2 {
  color: #2ecc71;
  font-size: 2.5rem;
  margin-bottom: 1.5rem;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.1);
}

.congratulations p {
  color: #34495e;
  font-size: 1.2rem;
}

.celebration-background {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 1000;
  background: url('@/assets/mouse.gif') center center;
  background-size: cover;
  opacity: 1;
}

.congratulations .password-display {
  background-color: #f8f9fa;
  padding: 1rem 1.5rem;
  border-radius: 10px;
  margin: 1.5rem 0;
  font-family: 'JMH Typewriter', monospace;
  word-break: break-all;
}

.congratulations .password-label {
  color: #7f8c8d;
  font-size: 0.9rem;
  margin-bottom: 0.5rem;
}

.congratulations .password-value {
  color: #2c3e50;
  font-size: 1.2rem;
  font-weight: bold;
}

.congratulations .time-wasted {
  color: #e74c3c;
  font-size: 1.3rem;
  margin-top: 1.5rem;
  font-weight: bold;
  text-transform: uppercase;
}

@media (max-width: 600px) {
  .content {
    padding: 2rem;
  }

  input {
    font-size: 14px;
    padding: 12px;
  }
  
  h1 {
    font-size: 1.8rem;
  }

  .rule-text {
    font-size: 0.85rem;
  }

  .congratulations {
    padding: 2rem;
    width: 90%;
  }

  .congratulations h2 {
    font-size: 2rem;
  }

  .congratulations .password-value {
    font-size: 1rem;
  }

  .congratulations .time-wasted {
    font-size: 1.1rem;
  }
}

@media (max-width: 400px) {
  .rule-text {
    font-size: 0.75rem;
  }
}
</style>

@keyframes popIn {
  0% {
    opacity: 0;
    transform: translate(-50%, -50%) scale(0.5);
  }
  100% {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1);
  }
}