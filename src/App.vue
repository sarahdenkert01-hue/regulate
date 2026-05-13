<script setup>
import { ref } from 'vue'

const currentState = ref(null)
const expandedTool = ref(null)
const favorites = ref([])
const showFeedback = ref(null)
const feedbackData = ref([])
const showStats = ref(false)
const showPanic = ref(false)
const panicMode = ref(null)
const vergenceStep = ref(0)
const vergenceComplete = ref(false)
const narration = ref('')
const currentArousal = ref(50)
const arousalHistory = ref([])
const showWindowOfTolerance = ref(false)
const apiKey = import.meta.env.VITE_ELEVENLABS_API_KEY
  
let currentAudio = null
  
function loadFavorites() {
  const saved = localStorage.getItem('regulate-favorites')
  if (saved) {
    favorites.value = JSON.parse(saved)
  }
}

function saveFavorites() {
  localStorage.setItem('regulate-favorites', JSON.stringify(favorites.value))
}

function toggleFavorite(toolId) {
  if (favorites.value.includes(toolId)) {
    favorites.value = favorites.value.filter(id => id !== toolId)
  } else {
    favorites.value.push(toolId)
  }
  saveFavorites()
}

function openFeedback(toolId) {
  showFeedback.value = toolId
}

function closeFeedback() {
  showFeedback.value = null
}

function submitFeedback(toolId, rating) {
  feedbackData.value.push({
    toolId,
    state: currentState.value,
    rating,
    timestamp: new Date()
  })
  localStorage.setItem('regulate-feedback', JSON.stringify(feedbackData.value))
  closeFeedback()
}

function getToolStats(state) {
  const stateData = feedbackData.value.filter(f => f.state === state)
  const toolStats = {}
  
  stateData.forEach(feedback => {
    if (!toolStats[feedback.toolId]) {
      toolStats[feedback.toolId] = { yes: 0, little: 0, no: 0, total: 0 }
    }
    toolStats[feedback.toolId][feedback.rating]++
    toolStats[feedback.toolId].total++
  })
  
  return toolStats
}

function getBestToolForState(state) {
  const stats = getToolStats(state)
  let bestTool = null
  let bestScore = -1
  
  Object.entries(stats).forEach(([toolId, data]) => {
    const score = (data.yes * 2 + data.little * 1) / data.total
    if (score > bestScore) {
      bestScore = score
      bestTool = toolId
    }
  })
  
  return bestTool
}

function getToolSuccessRate(toolId, state) {
  const stats = getToolStats(state)
  if (!stats[toolId]) return 0
  const data = stats[toolId]
  return Math.round((data.yes * 2 + data.little * 1) / (data.total * 2) * 100)
}

function getToolName(toolId) {
  for (let state of Object.values(states)) {
    const tool = state.tools.find(t => t.id === toolId)
    if (tool) return tool.name
  }
  return ''
}

function getTotalFeedback() {
  return feedbackData.value.length
}

function loadFeedback() {
  const saved = localStorage.getItem('regulate-feedback')
  if (saved) {
    feedbackData.value = JSON.parse(saved)
  }
}

const vergenceGuide = {
  quick: [
    { duration: 8, text: 'Find a spot about 4-6 inches from your face. Focus here.', animation: 'near' },
    { duration: 8, text: 'Now find a spot 10 feet away. Look there.', animation: 'far' },
    { duration: 8, text: 'Back to close. Feel the shift.', animation: 'near' },
    { duration: 8, text: 'Far again. Breathe.', animation: 'far' },
    { duration: 8, text: 'Near.', animation: 'near' },
    { duration: 8, text: 'Far. Keep going.', animation: 'far' },
    { duration: 8, text: 'Near. You\'re doing great.', animation: 'near' },
    { duration: 8, text: 'Far. Feel the calm.', animation: 'far' },
    { duration: 8, text: 'Near. You\'re safe.', animation: 'near' },
    { duration: 8, text: 'Far. Calm is returning.', animation: 'far' },
    { duration: 8, text: 'Near. Feel yourself grounding.', animation: 'near' },
    { duration: 8, text: 'Far. You\'re in control.', animation: 'far' },
    { duration: 8, text: 'Near, almost there', animation: 'near' },
    { duration: 8, text: 'Far. Take a deep breath.', animation: 'far' },
    { duration: 8, text: 'Near. You\'re safe.', animation: 'near' },
    { duration: 5, text: 'And... breathe. You\'re grounded.', animation: 'center' }
  ]
}

let currentAudio = null

async function speak(text) {
  try {
    console.log('API key:', import.meta.env.VITE_ELEVENLABS_API_KEY)
    console.log('Speaking:', text)

    if (!import.meta.env.VITE_ELEVENLABS_API_KEY) {
      throw new Error('Missing ElevenLabs API key. Check your .env file and restart npm run dev.')
    }

    if (currentAudio) {
      currentAudio.pause()
      currentAudio = null
    }

    const response = await fetch(
      'https://api.elevenlabs.io/v1/text-to-speech/YOUR_VOICE_ID?output_format=mp3_44100_128',
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'xi-api-key': import.meta.env.VITE_ELEVENLABS_API_KEY
        },
        body: JSON.stringify({
          text,
          model_id: 'eleven_multilingual_v2',
          voice_settings: {
            stability: 0.82,
            similarity_boost: 0.88,
            style: 0.12,
            use_speaker_boost: true
          }
        })
      }
    )

    console.log('ElevenLabs status:', response.status)

    if (!response.ok) {
      const errorText = await response.text()
      throw new Error(`ElevenLabs error ${response.status}: ${errorText}`)
    }

    const audioBlob = await response.blob()
    const audioUrl = URL.createObjectURL(audioBlob)

    currentAudio = new Audio(audioUrl)
    await currentAudio.play()
  } catch (error) {
    console.error('Speech error:', error)
  }
}

    const audioBlob = await response.blob()
    const audioUrl = URL.createObjectURL(audioBlob)

    currentAudio = new Audio(audioUrl)
    currentAudio.play()
  } catch (error) {
    console.error('Speech error:', error)
  }
}

    const audioBlob = await response.blob()
    const audioUrl = URL.createObjectURL(audioBlob)

    const audio = new Audio(audioUrl)
    audio.play()
  } catch (error) {
    console.error('Speech error:', error)
  }
}

async function startVergence(mode) {
  panicMode.value = mode
  vergenceStep.value = 0
  vergenceComplete.value = false
  
  const steps = vergenceGuide[mode]
  
  for (let i = 0; i < steps.length; i++) {
    vergenceStep.value = i
    narration.value = steps[i].text
    speak(steps[i].text)
    await new Promise(resolve => setTimeout(resolve, steps[i].duration * 1000))
  }
  
  vergenceComplete.value = true
}

function closePanic() {
  showPanic.value = false
  panicMode.value = null
  vergenceStep.value = 0
  vergenceComplete.value = false
  narration.value = ''
}

function handleFeelingBetter(better) {
  if (better) {
    closePanic()
  } else {
    vergenceComplete.value = false
    vergenceStep.value = 0
    narration.value = ''
  }
}

function updateArousal(value) {
  // Invert because vertical sliders work backwards
  currentArousal.value = 100 - parseInt(value)
}

function loadArousalHistory() {
  const saved = localStorage.getItem('regulate-arousal')
  if (saved) {
    arousalHistory.value = JSON.parse(saved)
  }
}

function getArousalStats() {
  if (arousalHistory.value.length === 0) return null
  
  const levels = arousalHistory.value.map(a => a.level)
  const average = Math.round(levels.reduce((a, b) => a + b, 0) / levels.length)
  const highest = Math.max(...levels)
  const lowest = Math.min(...levels)
  
  return { average, highest, lowest, total: levels.length }
}

function getArousalByState() {
  const byState = {}
  
  arousalHistory.value.forEach(entry => {
    if (!byState[entry.state]) {
      byState[entry.state] = []
    }
    byState[entry.state].push(entry.level)
  })
  
  const result = {}
  Object.entries(byState).forEach(([state, levels]) => {
    result[state] = Math.round(levels.reduce((a, b) => a + b, 0) / levels.length)
  })
  
  return result
}

const showSaveNotification = ref(false)

function saveArousalState() {
  arousalHistory.value.push({
    level: currentArousal.value,
    timestamp: new Date(),
    state: currentState.value
  })
  localStorage.setItem('regulate-arousal', JSON.stringify(arousalHistory.value))
  
  // Show notification
  showSaveNotification.value = true
  setTimeout(() => {
    showSaveNotification.value = false
  }, 2000)
}

loadFavorites()
loadFeedback()
loadArousalHistory()

const states = {
  anxious: {
    emoji: '😰',
    label: 'Overwhelmed / Anxious',
    tools: [
      {
        id: 'wall-push',
        icon: '🫸🏻',
        name: 'Push Against a Wall',
        instruction: 'Stand 12 inches from a wall. Push hard with both palms for 10 seconds.',
        why: 'Isometric pushing engages your proprioceptive system, telling your brain you\'re physically safe.'
      },
      {
        id: 'cold-water',
        icon: '💧',
        name: 'Cold Water on Wrists',
        instruction: 'Run cold water over the inside of your wrists for 30 seconds.',
        why: 'The vagus nerve runs near your wrist. Cold water triggers the dive reflex, slowing heart rate.'
      },
      {
        id: 'box-breath',
        icon: '🫁',
        name: '4-4-4 Breathing',
        instruction: 'Breathe in for 4 counts. Hold for 4. Out for 4. Repeat 3 times.',
        why: 'Extended exhales activate the parasympathetic nervous system.'
      }
    ]
  },
  numb: {
    emoji: '😶',
    label: 'Numb / Dissociated',
    tools: [
      {
        id: 'shake-arms',
        icon: '⌇🧍🏻‍♂️⌇',
        name: 'Shake Your Arms',
        instruction: 'Stand up. Shake both arms from the shoulder for 30 seconds.',
        why: 'Shaking is the body\'s natural way of discharging freeze energy.'
      },
      {
        id: 'self-compression',
        icon: '🙏',
        name: 'Self-Compression',
        instruction: 'Cross your arms and squeeze your shoulders or upper arms, hold for 15-30 seconds. Then press your palms firmly together, hold for 15-30 seconds.',
        why: 'Deep pressure gives your brain clear body feedback (proprioception), which helps reorient you to your physical self and reduces dissociation.'
      },
      {
        id: 'stomp-feet',
        icon: '👣',
        name: 'Stomp Your Feet',
        instruction: 'Stomp both feet on the floor, alternating left-right, for 30 seconds.',
        why: 'Bilateral stimulation activates both brain hemispheres simultaneously.'
      }
    ]
  },
  irritated: {
    emoji: '😤',
    label: 'Irritated / On Edge',
    tools: [
      {
        id: 'shoulder-shrug',
        icon: '🤷🏽',
        name: 'Shoulder Shrug & Roll',
        instruction: 'Lift your shoulders up towards your ears, hold for 5 seconds, then release. Slowly roll shoulders backwards 5-10 times.',
        why: 'Irritability often lives in muscular tension. Moving your shoulders and upper back engages the body and releases stored stress, signaling your nervous system that it\'s safe to relax.'
      },
      {
        id: 'cold-face',
        icon: '🧊',
        name: 'Cold Water on Face',
        instruction: 'Splash your face with cold water 3 times. Or hold a cold pack on your forehead.',
        why: 'The mammalian dive reflex slows heart rate within seconds.'
      },
      {
        id: 'long-exhale',
        icon: '😮‍💨',
        name: 'Sigh It Out',
        instruction: 'Take a normal breath in, then add a second short inhale on top. Let it all out slowly.',
        why: 'The physiological sigh is the fastest known way to reduce physiological arousal.'
      }
    ]
  },
  scattered: {
    emoji: '😵‍💫',
    label: 'Scattered / ADHD Overload',
    tools: [
      {
        id: 'one-tab',
        icon: '📌',
        name: 'One Tab Rule',
        instruction: 'Close everything except one thing. Type: "Right now I am working on ___." Then do that one thing.',
        why: 'Task-switching is cognitively expensive. External working memory helps your brain start.'
      },
      {
        id: 'tracing',
        icon: '🤚🏼',
        name: 'Trace Your Hand',
        instruction: 'Use pointer finger of one hand to trace the outline of your other hand. Then switch.',
        why: 'Engages fine motor touch and concentration, bringing your brain out of scattered loops into a focused sensory task. This is a somatic regulation strategy.'
      },
      {
        id: 'vergence',
        icon: '🚶👀',
        name: 'Vergence',
        instruction: 'Hold pointer finger about 4-6 inches from face. Focus on this spot for 3-5 seconds. Then find a spot about 10 feet away. Focus on this spot for 3-5 seconds. Alternate near to far repeatedly for 1-2 minutes.',
        why: 'By shifting gaze between near and far objects, it stimulates the vagus nerve and oculocardiac reflex, stimulating the parasympathetic system and reducing sympathetic "fight-or-flight" responses.'
      }
    ]
  },
  regulated: {
    emoji: '🙂',
    label: 'Regulated',
    tools: [
      {
        id: 'anchor-it',
        icon: '⚓️',
        name: 'Anchor This Feeling',
        instruction: 'Place one hand on your chest. Feel your heartbeat. Notice where in your body you feel calm.',
        why: 'Interoceptive awareness trains your nervous system to return to this state more easily.'
      },
      {
        id: 'build-buffer',
        icon: '🛡️',
        name: 'Build a Buffer',
        instruction: 'While you\'re regulated, do one thing to reduce tomorrow\'s load: lay out clothes, prep a meal.',
        why: 'Using this window for future planning genuinely reduces dysregulation tomorrow.'
      }
    ]
  }
}

function selectState(state) {
  currentState.value = state
}

function goBack() {
  currentState.value = null
}

function toggleWhy(toolId) {
  if (expandedTool.value === toolId) {
    expandedTool.value = null
  } else {
    expandedTool.value = toolId
  }
}
</script>

<template>
  <div class="app">
    <!-- Home Screen -->
    <div v-if="currentState === null && !showStats && !showWindowOfTolerance" class="screen">
      <div class="header">
        <div class="header-top">
          <h1>How is your nervous system?</h1>
          <button 
            class="stats-btn"
            @click="showStats = true"
            title="View insights"
          >
            📊
          </button>
        </div>
        <p>No judgment. Just check in.</p>
      </div>

      <!-- Favorites Section -->
      <div v-if="favorites.length > 0" class="favorites-section">
        <div class="section-label">⭐️ Your Favorites</div>
        <div class="favorites-grid">
          <div 
            v-for="toolId in favorites"
            :key="toolId"
            class="quick-tool-card"
          >
            <template v-for="state in Object.values(states)" :key="`state-${state.label}`">
              <template v-for="tool in state.tools" :key="tool.id">
                <div v-if="tool.id === toolId" class="tool-display">
                  <div class="quick-tool-emoji">{{ tool.icon }}</div>
                  <div class="quick-tool-name">{{ tool.name }}</div>
                </div>
              </template>
            </template>
          </div>
        </div>
      </div>

      <!-- Save Notification Toast -->
<transition name="fade">
  <div v-if="showSaveNotification" class="save-notification">
    ✓ Saved!
  </div>
</transition>

            <div class="section-label">😌 How are you feeling?</div>
      
      <div class="states-grid">
        <button 
          v-for="(data, key) in states" 
          :key="key"
          @click="selectState(key)"
          class="state-btn"
        >
          <span class="emoji">{{ data.emoji }}</span>
          <div class="state-info">
            <span class="label">{{ data.label }}</span>
          </div>
          <span class="arrow">›</span>
        </button>
      </div>

      <!-- Window of Tolerance Button -->
      <div class="wot-button-section">
        <button 
          class="wot-access-btn"
          @click="showWindowOfTolerance = true"
        >
          <div class="wot-btn-icon">🪟</div>
          <div class="wot-btn-text">
            <div class="wot-btn-title">Window of Tolerance</div>
            <div class="wot-btn-subtitle">Check your arousal level</div>
          </div>
          <span class="wot-btn-arrow">›</span>
        </button>
      </div>
    </div>

    <!-- Tools Screen -->
    <div v-else-if="currentState && !showStats" class="screen">
      <div class="tools-header">
        <button @click="goBack" class="back-btn">←</button>
        <div>
          <div class="state-tag">{{ states[currentState].emoji }} {{ states[currentState].label }}</div>
          <h2>Try One Of These</h2>
        </div>
      </div>

      <div class="tools-list">
        <div 
          v-for="tool in states[currentState].tools"
          :key="tool.id"
          class="tool-card"
        >
          <div class="tool-top">
            <div class="tool-icon">{{ tool.icon }}</div>
            <div class="tool-info">
              <div class="tool-name">{{ tool.name }}</div>
              <div class="tool-instruction">{{ tool.instruction }}</div>
            </div>
          </div>

          <div class="tool-actions">
            <button 
              class="why-btn"
              @click="toggleWhy(tool.id)"
            >
              💡 Why this works
            </button>
            <button 
              class="fav-btn"
              @click="toggleFavorite(tool.id)"
              :class="{ favorited: favorites.includes(tool.id) }"
            >
              {{ favorites.includes(tool.id) ? '⭐️' : '☆' }}
            </button>
          </div>

          <div v-if="expandedTool === tool.id" class="why-content">
            {{ tool.why }}
          </div>

          <button class="do-it-btn" @click="openFeedback(tool.id)">
            I'll try this →
          </button>
        </div>
      </div>

      <!-- Feedback Modal -->
      <div v-if="showFeedback" class="feedback-overlay">
        <div class="feedback-modal">
          <div class="feedback-header">
            <h3>Did this help?</h3>
            <button class="feedback-close" @click="closeFeedback">✕</button>
          </div>
          
          <div class="feedback-buttons">
            <button 
              class="feedback-btn yes-btn"
              @click="submitFeedback(showFeedback, 'yes')"
            >
              <div class="feedback-emoji">👍</div>
              <div class="feedback-label">Yes!</div>
            </button>
            
            <button 
              class="feedback-btn little-btn"
              @click="submitFeedback(showFeedback, 'little')"
            >
              <div class="feedback-emoji">🤷</div>
              <div class="feedback-label">A Little</div>
            </button>
            
            <button 
              class="feedback-btn no-btn"
              @click="submitFeedback(showFeedback, 'no')"
            >
              <div class="feedback-emoji">👎</div>
              <div class="feedback-label">No</div>
            </button>
          </div>
        </div>
      </div>

      <!-- Panic Button -->
      <button 
        class="panic-button"
        @click="showPanic = true"
        title="Guided Brainspotting"
      >
        🆘
      </button>
    </div>

    <!-- Stats Screen -->
    <div v-else-if="showStats" class="screen">
      <div class="stats-header">
        <button @click="showStats = false" class="back-btn">←</button>
        <div>
          <h2>Your Insights 📊</h2>
        </div>
      </div>

      <div class="stats-content">
        <div class="stat-card">
          <div class="stat-number">{{ getTotalFeedback() }}</div>
          <div class="stat-label">Tools Used</div>
        </div>
      
      <!-- Arousal Stats -->
<div v-if="getArousalStats()" class="arousal-stats-section">
  <div class="arousal-header">
    <h3>Nervous System Insights 🧠</h3>
  </div>

  <div class="arousal-grid">
    <div class="arousal-stat-card">
      <div class="arousal-label">Average Arousal</div>
      <div class="arousal-value">{{ getArousalStats().average }}</div>
      <div class="arousal-subtext">
        <span v-if="getArousalStats().average < 33" class="hyper-text">Hyperarousal zone</span>
        <span v-else-if="getArousalStats().average < 67" class="regulated-text">Window zone</span>
        <span v-else class="hypo-text">Hypoarousal zone</span>
      </div>
    </div>

    <div class="arousal-stat-card">
      <div class="arousal-label">Highest</div>
      <div class="arousal-value">{{ getArousalStats().highest }}</div>
      <div class="arousal-subtext">Most activated</div>
    </div>

    <div class="arousal-stat-card">
      <div class="arousal-label">Lowest</div>
      <div class="arousal-value">{{ getArousalStats().lowest }}</div>
      <div class="arousal-subtext">Most deactivated</div>
    </div>

    <div class="arousal-stat-card">
      <div class="arousal-label">Check-ins</div>
      <div class="arousal-value">{{ getArousalStats().total }}</div>
      <div class="arousal-subtext">Times tracked</div>
    </div>
  </div>

  <!-- By State -->
  <div class="arousal-by-state">
    <div class="arousal-by-state-title">Average by Nervous State</div>
    <div v-for="(stateData, stateKey) in states" :key="stateKey" class="state-arousal">
      <div class="state-arousal-header">
        <span class="state-emoji">{{ stateData.emoji }}</span>
        <span class="state-name">{{ stateData.label }}</span>
      </div>
      <div v-if="getArousalByState()[stateKey]" class="state-arousal-value">
        {{ getArousalByState()[stateKey] }}
      </div>
      <div v-else class="state-arousal-empty">
        No data
      </div>
    </div>
  </div>
</div>

        <div v-for="(stateData, stateKey) in states" :key="stateKey" class="state-stats">
          <div class="state-stats-header">
            {{ stateData.emoji }} {{ stateData.label }}
          </div>

          <div v-if="getToolStats(stateKey) && Object.keys(getToolStats(stateKey)).length > 0" class="tools-stats">
            <div 
              v-for="(stats, toolId) in getToolStats(stateKey)"
              :key="toolId"
              class="tool-stat-card"
            >
              <div class="tool-stat-top">
                <div class="tool-stat-name">{{ getToolName(toolId) }}</div>
                <div class="tool-stat-rate">{{ getToolSuccessRate(toolId, stateKey) }}%</div>
              </div>
              
              <div class="tool-stat-feedback">
                <div class="feedback-count yes">
                  <span class="emoji">👍</span> {{ stats.yes }}
                </div>
                <div class="feedback-count little">
                  <span class="emoji">🤷</span> {{ stats.little }}
                </div>
                <div class="feedback-count no">
                  <span class="emoji">👎</span> {{ stats.no }}
                </div>
              </div>
              
              <div class="progress-bar">
                <div 
                  class="progress-fill"
                  :style="{ width: (getToolSuccessRate(toolId, stateKey)) + '%' }"
                ></div>
              </div>
            </div>
          </div>
          
          <div v-else class="no-data">
            No data yet. Try some tools!
          </div>
        </div>
      </div>
    </div>

    <!-- Window of Tolerance Screen -->
    <div v-else-if="showWindowOfTolerance" class="screen">
      <div class="tools-header">
        <button @click="showWindowOfTolerance = false" class="back-btn">←</button>
        <div>
          <h2>Where are you right now?</h2>
        </div>
      </div>

      <div class="wot-content">
        <!-- Full Visual -->
        <div class="wot-visual">
          <!-- Top Zone: Hyperarousal -->
          <div class="wot-zone-top">
            <div class="zone-side-text">
              <div class="zone-arrow">↑</div>
              <div class="zone-title">Hyperarousal</div>
            </div>
            <div class="zone-description-top">
              <strong>Activated, Fight or Flight</strong>
              <p>Anger, fear, anxiety, panic, hypervigilance</p>
            </div>
          </div>

          <!-- Middle: Window of Tolerance with Frame -->
          <div class="wot-middle">
            <div class="wot-window-frame">
              <div class="window-pane top-left"></div>
              <div class="window-pane top-right"></div>
              <div class="window-pane bottom-left"></div>
              <div class="window-pane bottom-right"></div>
              
              <!-- Content Inside Window -->
              <div class="window-content">
                <div class="window-title">Optimal Zone</div>
                <p>When you are inside your window of tolerance, you feel regulated, stable, and calm.</p>
              </div>
            </div>

            <!-- Slider on the right side -->
            <div class="wot-slider-container">
              <div class="slider-numbers">
                <div class="slider-num top-num">0</div>
                <div class="slider-num mid-num">50</div>
                <div class="slider-num bottom-num">100</div>
              </div>

              <input 
                type="range" 
                min="0" 
                max="100" 
                :value="100 - currentArousal"
                @input="updateArousal($event.target.value)"
                class="wot-slider"
              />
              <div 
                class="wot-marker"
                :style="{ '--marker-pos': (currentArousal / 100) * 240 }"
              >
                <div class="marker-dot"></div>
                <div class="marker-value">{{ currentArousal }}</div>
              </div>
            </div>
          </div>

          <!-- Bottom Zone: Hypoarousal -->
          <div class="wot-zone-bottom">
            <div class="zone-description-bottom">
              <strong>Deactivated, Freeze or Fawn</strong>
              <p>Detachment, shut down, depression, absence of sensations</p>
            </div>
            <div class="zone-side-text">
              <div class="zone-title">Hypoarousal</div>
              <div class="zone-arrow">↓</div>
            </div>
          </div>
        </div>
      </div>

      <!-- Current State Display -->
      <div class="wot-state">
        <div v-if="currentArousal < 33" class="state-box hyper">
          <div class="state-emoji">🚨</div>
          <div class="state-text">You're in Hyperarousal</div>
          <div class="state-advice">Try calming tools like cold water or breathing</div>
        </div>
        <div v-else-if="currentArousal < 67" class="state-box regulated">
          <div class="state-emoji">✨</div>
          <div class="state-text">You're in your Window of Tolerance</div>
          <div class="state-advice">You're doing well! Use this moment to anchor this feeling</div>
        </div>
        <div v-else class="state-box hypo">
          <div class="state-emoji">😶</div>
          <div class="state-text">You're in Hypoarousal</div>
          <div class="state-advice">Try activating tools like shaking or movement</div>
        </div>
      </div>

      <button class="wot-save-btn" @click="saveArousalState">
        📌 Save this moment
      </button>

      <!-- Save Notification Toast -->
      <transition name="fade">
        <div v-if="showSaveNotification" class="save-notification">
          ✓ Saved!
        </div>
      </transition>
    </div>

    <!-- Panic Screen -->
    <div v-if="showPanic" class="panic-screen">
      <div v-if="!panicMode" class="panic-intro">
        <div class="panic-header">
          <h1>Brainspotting Vergence</h1>
          <p>A grounding technique to calm your nervous system</p>
        </div>
        
        <div class="panic-buttons">
          <button class="panic-option quick" @click="startVergence('quick')">
            <div class="panic-time">2 min</div>
            <div class="panic-label">Quick</div>
          </button>
        </div>
        
        <button class="panic-close" @click="closePanic">Close</button>
      </div>

      <div v-else-if="!vergenceComplete" class="panic-guide">
        <button class="panic-exit" @click="closePanic">✕</button>
        
        <div class="vergence-container">
          <div class="vergence-animation">
            <div 
              class="vergence-dot"
              :class="{
                'near': vergenceGuide[panicMode][vergenceStep].animation === 'near',
                'far': vergenceGuide[panicMode][vergenceStep].animation === 'far',
                'breathe': vergenceGuide[panicMode][vergenceStep].animation === 'breathe',
                'center': vergenceGuide[panicMode][vergenceStep].animation === 'center'
              }"
            ></div>
          </div>
          
          <div class="narration">
            {{ narration }}
          </div>
          
          <div class="progress-bar">
            <div 
              class="progress-fill"
              :style="{ width: (vergenceStep / vergenceGuide[panicMode].length * 100) + '%' }"
            ></div>
          </div>
        </div>
      </div>

      <div v-else class="panic-complete">
        <div class="complete-content">
          <div class="complete-emoji">✨</div>
          <h2>Do you feel better?</h2>
          
          <div class="complete-buttons">
            <button class="complete-yes-btn" @click="handleFeelingBetter(true)">
              Yes 👍
            </button>
            <button class="complete-no-btn" @click="handleFeelingBetter(false)">
              Repeat? 🔄
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app {
  max-width: 430px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Red Hat Display', sans-serif;
  min-height: 100vh;
}

.screen {
  animation: fadeUp 0.3s ease;
}

@keyframes fadeUp {
  from { opacity: 0; transform: translateY(16px); }
  to { opacity: 1; transform: translateY(0); }
}

.header {
  text-align: center;
  margin-bottom: 40px;
  margin-top: 30px;
}

.header-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 10px;
}

.header h1 {
  font-size: 28px;
  font-weight: 800;
  margin-bottom: 0;
  color: #1a1714;
}

.header p {
  color: #6b6560;
  font-size: 15px;
}

.stats-btn {
  background: linear-gradient(135deg, #fff9e6 0%, #fffef0 100%);
  border: 2px solid #ffd700;
  border-radius: 16px;
  width: 44px;
  height: 44px;
  font-size: 20px;
  cursor: pointer;
  transition: all 0.2s;
  flex-shrink: 0;
}

.stats-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(255, 215, 0, 0.3);
}

.states-grid {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 100px;
}

.state-btn {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 18px 22px;
  border-radius: 20px;
  border: none;
  background: #fff;
  cursor: pointer;
  box-shadow: 0 4px 24px rgba(0,0,0,0.07);
  transition: all 0.25s cubic-bezier(0.34, 1.56, 0.64, 1);
  text-align: left;
}

.state-btn:hover {
  transform: translateY(-2px) scale(1.01);
  box-shadow: 0 8px 40px rgba(0,0,0,0.12);
}

.state-btn:active {
  transform: scale(0.98);
}

.emoji {
  font-size: 28px;
  flex-shrink: 0;
}

.state-info {
  flex: 1;
}

.label {
  font-size: 16px;
  font-weight: 700;
  color: #1a1714;
  display: block;
}

.arrow {
  font-size: 18px;
  color: #030303;
}

.tools-header {
  display: flex;
  align-items: flex-start;
  gap: 16px;
  margin-bottom: 30px;
  margin-top: 30px;
}

.back-btn {
  background: linear-gradient(135deg, #b8d4c8 0%, #c5d0d8 100%);
  border: none;
  border-radius: 12px;
  width: 40px;
  height: 40px;
  cursor: pointer;
  font-size: 18px;
  transition: background 0.2s;
  flex-shrink: 0;
}

.back-btn:hover {
  background: #e0d9d0;
}

.state-tag {
  display: inline-block;
  font-size: 20px;
  font-weight: 500;
  color: #4c443e;
  margin-bottom: 8px;
}

.tools-header h2 {
  font-size: 22px;
  font-weight: 800;
  color: #222222;
  margin: 0;
}

.tools-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 100px;
}

.tool-card {
  background: #fff;
  border-radius: 20px;
  padding: 20px 22px;
  box-shadow: 0 4px 24px rgba(0,0,0,0.07);
  transition: transform 0.25s, box-shadow 0.2s;
  width: 95%;
  margin: 0 auto;
}

.tool-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 40px rgba(0,0,0,0.12);
}

.tool-top {
  display: flex;
  align-items: flex-start;
  gap: 14px;
  margin-bottom: 12px;
}

.tool-icon {
  font-size: 32px;
  width: 44px;
  height: 44px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.tool-info {
  flex: 1;
}

.tool-name {
  font-size: 22px;
  font-weight: 700;
  color: #1a1714;
  margin-bottom: 4px;
  display: block;
}

.tool-instruction {
  font-size: 18px;
  color: #504b47;
  line-height: 1.5;
  font-weight: 500;
}

.tool-actions {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 12px;
}

.why-btn {
  font-size: 16px !important;
  font-weight: 700 !important;
  color: hsl(0, 0%, 100%) !important;
  background: hwb(216 57% 5%) !important;
  border: none;
  padding: 5px 12px;
  border-radius: 100px;
  cursor: pointer;
  font-family: 'Red Hat Display', sans-serif;
  transition: background 0.2s;
}

.why-btn:hover {
  background: #ede8e1;
}

.fav-btn {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  transition: transform 0.25s, color 0.2s;
  padding: 0;
  line-height: 1;
  color: #a09990;
}

.fav-btn:hover {
  transform: scale(1.3);
}

.fav-btn.favorited {
  color: #ffd700;
}

.why-content {
  font-size: 13px;
  color: #6b6560;
  line-height: 1.6;
  font-weight: 300;
  margin-bottom: 12px;
  padding: 12px 0;
  border-top: 1px solid #ede8e1;
  animation: fadeUp 0.2s ease;
}

.do-it-btn {
  width: 100%;
  padding: 16px;
  border-radius: 12px;
  border: none;
  font-family: 'Red Hat Display', sans-serif;
  font-size: 15px;
  font-weight: 700;
  cursor: pointer;
  background: hsl(153, 19%, 65%);
  color: white;
  transition: transform 0.25s, opacity 0.2s;
}

.do-it-btn:hover {
  transform: scale(1.02);
}

.do-it-btn:active {
  transform: scale(0.98);
  opacity: 0.85;
}

.section-label {
  font-size: 12px;
  font-weight: 600;
  color: #a09990;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 12px;
  margin-top: 24px;
}

.favorites-section {
  margin-bottom: 30px;
}

.favorites-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
}

.quick-tool-card {
  background: linear-gradient(135deg, #fff9e6 0%, #fffef0 100%);
  border-radius: 16px;
  padding: 16px;
  text-align: center;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
  border: 2px solid #ffd700;
}

.quick-tool-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 20px rgba(255, 215, 0, 0.3);
}

.tool-display {
  width: 100%;
}

.quick-tool-emoji {
  font-size: 32px;
  margin-bottom: 8px;
}

.quick-tool-name {
  font-size: 12px;
  font-weight: 700;
  color: #1a1714;
  line-height: 1.4;
}

.feedback-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: flex-end;
  z-index: 1000;
  animation: fadeIn 0.2s ease;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.feedback-modal {
  background: white;
  border-radius: 24px 24px 0 0;
  padding: 28px 24px;
  width: 100%;
  max-width: 430px;
  margin: 0 auto;
  animation: slideUp 0.3s ease;
}

@keyframes slideUp {
  from { transform: translateY(100%); }
  to { transform: translateY(0); }
}

.feedback-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 24px;
}

.feedback-header h3 {
  font-size: 20px;
  font-weight: 800;
  color: #1a1714;
  margin: 0;
}

.feedback-close {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #a09990;
  transition: color 0.2s;
}

.feedback-close:hover {
  color: #1a1714;
}

.feedback-buttons {
  display: flex;
  gap: 12px;
  justify-content: center;
}

.feedback-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  padding: 16px 20px;
  border-radius: 16px;
  border: 2px solid transparent;
  cursor: pointer;
  transition: all 0.2s;
  flex: 1;
  background: #f5f2ee;
}

.feedback-btn:hover {
  transform: translateY(-2px);
}

.yes-btn {
  border-color: #4a7c6f;
  background: rgba(74, 124, 111, 0.1);
}

.yes-btn:hover {
  background: rgba(74, 124, 111, 0.2);
}

.little-btn {
  border-color: #c4a861;
  background: rgba(196, 168, 97, 0.1);
}

.little-btn:hover {
  background: rgba(196, 168, 97, 0.2);
}

.no-btn {
  border-color: #c4746e;
  background: rgba(196, 116, 110, 0.1);
}

.no-btn:hover {
  background: rgba(196, 116, 110, 0.2);
}

.feedback-emoji {
  font-size: 28px;
}

.feedback-label {
  font-size: 12px;
  font-weight: 700;
  color: #1a1714;
}

/* WINDOW OF TOLERANCE */
.wot-section {
  margin-bottom: 40px;
  margin-top: 30px;
}

.wot-header {
  text-align: center;
  margin-bottom: 24px;
}

.wot-header h3 {
  font-size: 20px;
  font-weight: 800;
  color: rgb(55, 55, 55);
  margin: 0 0 8px 0;
}

.wot-header p {
  font-size: 13px;
  color: #6b6560;
  margin: 0;
}

.wot-container {
  background: linear-gradient(135deg, #b8d4c8 0%, #c5d0d8 100%);
  border-radius: 20px;
  padding: 20px;
  box-shadow: 0 4px 24px rgba(0,0,0,0.07);
}

.wot-visual {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.wot-zone-top {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px;
  background: linear-gradient(135deg, #d4b8cb 0%, #c5d0d8 100%);
  border-radius: 12px;
}

.zone-side-text {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  font-weight: 700;
  color: rgb(55, 55, 55);
  writing-mode: vertical-rl;
  text-orientation: mixed;
}

.zone-arrow {
  font-size: 20px;
}

.zone-title {
  font-size: 12px;
  letter-spacing: 1px;
}

.zone-description-top {
  flex: 1;
}

.zone-description-top strong {
  display: block;
  font-size: 13px;
  font-weight: 700;
  color: rgb(55, 55, 55);
  margin-bottom: 4px;
}

.zone-description-top p {
  font-size: 12px;
  color: #555;
  margin: 0;
  line-height: 1.4;
}

.wot-middle {
  display: flex;
  gap: 16px;
  align-items: center;
  justify-content: center;
}

.wot-window-frame {
  position: relative;
  width: 220px;
  height: 240px;
  background: #c5c8d87c;
  border: 8px solid #1a1714;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 16px;
  text-align: center;
}

.window-pane {
  position: absolute;
  border: 2px solid #1a1714;
  background: transparent;
}

.window-pane.top-left {
  top: 8px;
  left: 8px;
  width: 45%;
  height: 40%;
  border-right: none;
  border-bottom: none;
}

.window-pane.top-right {
  top: 8px;
  right: 8px;
  width: 45%;
  height: 40%;
  border-left: none;
  border-bottom: none;
}

.window-pane.bottom-left {
  bottom: 8px;
  left: 8px;
  width: 45%;
  height: 40%;
  border-right: none;
  border-top: none;
}

.window-pane.bottom-right {
  bottom: 8px;
  right: 8px;
  width: 45%;
  height: 40%;
  border-left: none;
  border-top: none;
}

.window-content {
  position: relative;
  z-index: 10;
}

.window-title {
  font-size: 16px;
  font-weight: 800;
  color:rgb(55, 55, 55);
  margin-bottom: 8px;
}

.window-content p {
  font-size: 12px;
  color: #555;
  line-height: 1.4;
  margin: 0;
}

.wot-slider-container {
  position: relative;
  width: 60px;
  height: 240px;
  background: linear-gradient(to bottom, #ffcdd2 0%, #a5d6a7 50%, #bbdefb 100%);
  border-radius: 30px;
  border: 3px solid #1a1714;
  overflow: hidden;
  margin-right: 40px;
}

.wot-slider {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
  cursor: pointer;
  z-index: 10;
  appearance: slider-vertical;
  -webkit-appearance: slider-vertical;
  writing-mode: bt-lr;
}

.wot-marker {
  position: absolute;
  left: 50%;
  top: 0;
  transform: translateX(-50%) translateY(calc(var(--marker-pos) * 1px - 28px));
  transition: transform 0.2s ease;
  z-index: 20;
  pointer-events: none;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}

.marker-dot {
  width: 56px;
  height: 56px;
  border-radius: 50%;
  background: white;
  border: 4px solid #1a1714;
  box-shadow: 0 4px 12px rgba(0,0,0,0.3);
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
}

.wot-zone-bottom {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px;
  background: linear-gradient(135deg, #c5c8d8 0%, #b8cfd4 100%);
  border-radius: 12px;
}

.zone-description-bottom {
  flex: 1;
}

.zone-description-bottom strong {
  display: block;
  font-size: 13px;
  font-weight: 700;
  color: #1a1714;
  margin-bottom: 4px;
}

.zone-description-bottom p {
  font-size: 12px;
  color: #555;
  margin: 0;
  line-height: 1.4;
}

.wot-state {
  margin-top: 20px;
  margin-bottom: 16px;
}

.state-box {
  border-radius: 16px;
  padding: 16px;
  text-align: center;
  animation: slideUp 0.3s ease;
}

.state-box.hyper {
  background: rgba(255, 193, 7, 0.1);
  border: 2px solid #ff6b6b;
}

.state-box.regulated {
  background: rgba(74, 124, 111, 0.1);
  border: 2px solid #4a7c6f;
}

.state-box.hypo {
  background: rgba(33, 150, 243, 0.1);
  border: 2px solid #2196f3;
}

.state-emoji {
  font-size: 32px;
  margin-bottom: 8px;
}

.state-text {
  font-size: 14px;
  font-weight: 700;
  color: #1a1714;
  margin-bottom: 4px;
}

.state-advice {
  font-size: 12px;
  color: #6b6560;
  line-height: 1.4;
}

.wot-save-btn {
  width: 100%;
  padding: 12px;
  border-radius: 12px;
  border: none;
  background: linear-gradient(135deg, #b8d4c8 0%, #c5d0d8 100%);
  color: rgb(55, 55, 55);
  font-size: 14px;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s;
  font-family: 'Red Hat Display', sans-serif;
}

.wot-save-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 20px rgba(126, 198, 159, 0.3);
}

.wot-save-btn:active {
  transform: scale(0.98);
}

/* STATS SCREEN */
.stats-header {
  display: flex;
  align-items: flex-start;
  gap: 16px;
  margin-bottom: 30px;
  margin-top: 30px;
}

.stats-header h2 {
  font-size: 22px;
  font-weight: 800;
  color: #222222;
  margin: 0;
}

.stats-content {
  display: flex;
  flex-direction: column;
  gap: 20px;
  margin-bottom: 100px;
}

.stat-card {
  background: linear-gradient(135deg, #a8d5ba 0%, #7ec69f 100%);
  border-radius: 20px;
  padding: 28px 24px;
  text-align: center;
  color: white;
}

.stat-number {
  font-size: 48px;
  font-weight: 800;
  margin-bottom: 8px;
}

.stat-label {
  font-size: 14px;
  font-weight: 600;
  opacity: 0.9;
}

.state-stats {
  margin-top: 24px;
}

.state-stats-header {
  font-size: 16px;
  font-weight: 700;
  color: #1a1714;
  margin-bottom: 12px;
  padding-bottom: 8px;
  border-bottom: 2px solid #ede8e1;
}

.tools-stats {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.tool-stat-card {
  background: #fff;
  border-radius: 16px;
  padding: 16px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.05);
}

.tool-stat-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.tool-stat-name {
  font-size: 14px;
  font-weight: 700;
  color: #1a1714;
}

.tool-stat-rate {
  font-size: 18px;
  font-weight: 800;
  color: #4a7c6f;
}

.tool-stat-feedback {
  display: flex;
  gap: 12px;
  margin-bottom: 12px;
  font-size: 12px;
}

.feedback-count {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 4px 8px;
  border-radius: 8px;
  background: #f5f2ee;
}

.feedback-count.yes {
  background: rgba(74, 124, 111, 0.1);
}

.feedback-count.little {
  background: rgba(196, 168, 97, 0.1);
}

.feedback-count.no {
  background: rgba(196, 116, 110, 0.1);
}

.no-data {
  text-align: center;
  color: #a09990;
  font-size: 14px;
  padding: 20px;
}

/* PANIC BUTTON */
.panic-button {
  position: fixed;
  bottom: 30px;
  right: 30px;
  width: 70px;
  height: 70px;
  border-radius: 50%;
  border: none;
  background: linear-gradient(135deg, #ff6b6b 0%, #ee5a6f 100%);
  font-size: 32px;
  cursor: pointer;
  box-shadow: 0 8px 24px rgba(255, 107, 107, 0.4);
  transition: all 0.3s;
  z-index: 500;
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.05); }
}

.panic-button:hover {
  transform: scale(1.1);
  box-shadow: 0 12px 32px rgba(255, 107, 107, 0.6);
}

.panic-button:active {
  transform: scale(0.95);
}

.panic-screen {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, #e8f4f8 0%, #f0e8f8 100%);
  z-index: 2000;
  display: flex;
  align-items: center;
  justify-content: center;
  animation: fadeIn 0.3s ease;
}

.panic-intro {
  text-align: center;
  max-width: 350px;
  animation: slideUp 0.5s ease;
}

.panic-header {
  margin-bottom: 40px;
}

.panic-header h1 {
  font-size: 32px;
  font-weight: 800;
  color: #1a1714;
  margin-bottom: 12px;
}

.panic-header p {
  font-size: 15px;
  color: #6b6560;
}

.panic-buttons {
  display: flex;
  gap: 16px;
  margin-bottom: 30px;
}

.panic-option {
  flex: 1;
  padding: 28px 20px;
  border-radius: 20px;
  border: none;
  cursor: pointer;
  transition: all 0.2s;
  font-family: 'Red Hat Display', sans-serif;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.panic-option.quick {
  background: linear-gradient(135deg, #a8d5ba 0%, #7ec69f 100%);
  color: white;
}

.panic-option.quick:hover {
  transform: translateY(-4px);
  box-shadow: 0 12px 24px rgba(126, 198, 159, 0.3);
}

.panic-time {
  font-size: 24px;
  font-weight: 800;
}

.panic-label {
  font-size: 14px;
  font-weight: 600;
}

.panic-close {
  background: none;
  border: none;
  color: #6b6560;
  font-size: 16px;
  cursor: pointer;
  padding: 12px 24px;
  border-radius: 12px;
  transition: all 0.2s;
}

.panic-close:hover {
  background: rgba(0, 0, 0, 0.05);
  color: #1a1714;
}

.panic-guide {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  position: relative;
  padding: 40px 20px;
}

.panic-exit {
  position: absolute;
  top: 20px;
  right: 20px;
  background: rgba(255, 255, 255, 0.9);
  border: none;
  width: 48px;
  height: 48px;
  border-radius: 50%;
  font-size: 24px;
  cursor: pointer;
  transition: all 0.2s;
}

.panic-exit:hover {
  background: white;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.vergence-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 40px;
  max-width: 400px;
}

.vergence-animation {
  width: 100%;
  height: 300px;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
}

.vergence-dot {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: linear-gradient(135deg, #a8d5ba 0%, #7ec69f 100%);
  box-shadow: 0 8px 24px rgba(126, 198, 159, 0.3);
  transition: all 0.6s cubic-bezier(0.34, 1.56, 0.64, 1);
}

.vergence-dot.near {
  transform: translate(0, -80px) scale(1.2);
}

.vergence-dot.far {
  transform: translate(0, 80px) scale(0.6);
  opacity: 0.6;
}

.vergence-dot.breathe {
  animation: breathePulse 3s infinite;
}

.vergence-dot.center {
  transform: scale(1);
}

@keyframes breathePulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.3); }
}

.narration {
  font-size: 20px;
  font-weight: 600;
  color: #1a1714;
  text-align: center;
  line-height: 1.6;
  min-height: 60px;
  animation: fadeInText 0.5s ease;
}

@keyframes fadeInText {
  from { opacity: 0; }
  to { opacity: 1; }
}

.panic-complete {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  height: 100%;
  animation: fadeIn 0.3s ease;
}

.complete-content {
  text-align: center;
  animation: slideUp 0.5s ease;
}

.complete-emoji {
  font-size: 80px;
  margin-bottom: 24px;
  animation: bounce 0.6s ease;
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-20px); }
}

.complete-content h2 {
  font-size: 28px;
  font-weight: 800;
  color: #1a1714;
  margin-bottom: 32px;
}

.complete-buttons {
  display: flex;
  gap: 16px;
  flex-direction: column;
}

.complete-yes-btn,
.complete-no-btn {
  padding: 16px 32px;
  border: none;
  border-radius: 16px;
  font-size: 16px;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s;
  font-family: 'Red Hat Display', sans-serif;
}

.complete-yes-btn {
  background: linear-gradient(135deg, #a8d5ba 0%, #7ec69f 100%);
  color: white;
}

.complete-yes-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 20px rgba(126, 198, 159, 0.3);
}

.complete-no-btn {
  background: linear-gradient(135deg, #b4a7d6 0%, #9088c9 100%);
  color: white;
}

.complete-no-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 20px rgba(144, 136, 201, 0.3);
}

.arousal-stats-section {
  margin-top: 24px;
}

.arousal-header {
  margin-bottom: 16px;
}

.arousal-header h3 {
  font-size: 16px;
  font-weight: 800;
  color: #1a1714;
  margin: 0;
}

.arousal-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
  margin-bottom: 24px;
}

.arousal-stat-card {
  background: #fff;
  border-radius: 16px;
  padding: 16px;
  text-align: center;
  box-shadow: 0 4px 12px rgba(0,0,0,0.05);
}

.arousal-label {
  font-size: 12px;
  font-weight: 600;
  color: #6b6560;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 8px;
}

.arousal-value {
  font-size: 32px;
  font-weight: 800;
  color: #1a1714;
  margin-bottom: 4px;
}

.arousal-subtext {
  font-size: 11px;
  color: #a09990;
}

.hyper-text {
  color: #ff6b6b;
  font-weight: 700;
}

.regulated-text {
  color: #4a7c6f;
  font-weight: 700;
}

.hypo-text {
  color: #2196f3;
  font-weight: 700;
}

.arousal-by-state {
  background: #fff;
  border-radius: 16px;
  padding: 16px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.05);
}

.arousal-by-state-title {
  font-size: 14px;
  font-weight: 700;
  color: #1a1714;
  margin-bottom: 12px;
  padding-bottom: 12px;
  border-bottom: 2px solid #ede8e1;
}

.state-arousal {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 0;
}

.state-arousal-header {
  display: flex;
  align-items: center;
  gap: 8px;
}

.state-emoji {
  font-size: 20px;
}

.state-name {
  font-size: 13px;
  font-weight: 600;
  color: #1a1714;
}

.state-arousal-value {
  font-size: 18px;
  font-weight: 800;
  color: #4a7c6f;
}

.state-arousal-empty {
  font-size: 12px;
  color: #a09990;
}

.progress-bar {
  width: 100%;
  height: 6px;
  background: #ede8e1;
  border-radius: 3px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #4a7c6f 0%, #7ec69f 100%);
  transition: width 0.3s ease;
}

.slider-numbers {
  position: absolute;
  right: -45px;
  top: 0;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  z-index: 5;
  padding: 8px 0;
}

.slider-num {
  font-size: 14px;
  font-weight: 800;
  color: #1a1714;
  width: 30px;
  text-align: left;
  padding-left: 8px;
}

/* Removed empty rulesets for .top-num, .mid-num, and .bottom-num */

.marker-value {
  font-size: 20px;
  font-weight: 800;
  color: #1a1714;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  margin: 0;
  pointer-events: none;
}

/* Toast Notification */
.save-notification {
  position: fixed;
  bottom: 100px;
  left: 50%;
  transform: translateX(-50%);
  background: linear-gradient(135deg, #4a7c6f 0%, #7ec69f 100%);
  color: white;
  padding: 16px 24px;
  border-radius: 50px;
  box-shadow: 0 8px 24px rgba(0,0,0,0.2);
  font-weight: 700;
  font-size: 14px;
  z-index: 3000;
  animation: slideUpToast 0.3s ease, slideDownToast 0.3s ease 1.7s;
  display: flex;
  align-items: center;
  gap: 8px;
}

@keyframes slideUpToast {
  from {
    transform: translateX(-50%) translateY(20px);
    opacity: 0;
  }
  to {
    transform: translateX(-50%) translateY(0);
    opacity: 1;
  }
}

@keyframes slideDownToast {
  from {
    transform: translateX(-50%) translateY(0);
    opacity: 1;
  }
  to {
    transform: translateX(-50%) translateY(20px);
    opacity: 0;
  }
}

.wot-button-section {
  margin-bottom: 30px;
  margin-top: 20px;
}

.wot-access-btn {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 18px 22px;
  border-radius: 20px;
  border: none;
  background: linear-gradient(135deg, #b8d4c8 0%, #c5d0d8 100%);
  cursor: pointer;
  box-shadow: 0 4px 24px rgba(0,0,0,0.07);
  transition: all 0.25s cubic-bezier(0.34, 1.56, 0.64, 1);
  text-align: left;
  width: 100%;
  font-family: 'Red Hat Display', sans-serif;
}

.wot-access-btn:hover {
  transform: translateY(-2px) scale(1.01);
  box-shadow: 0 8px 40px rgba(0,0,0,0.12);
}

.wot-access-btn:active {
  transform: scale(0.98);
}

.wot-btn-icon {
  font-size: 32px;
  flex-shrink: 0;
}

.wot-btn-text {
  flex: 1;
}

.wot-btn-title {
  font-size: 16px;
  font-weight: 700;
  color: #1a1714;
  margin-bottom: 4px;
}

.wot-btn-subtitle {
  font-size: 12px;
  color: #6b6560;
}

.wot-btn-arrow {
  font-size: 18px;
  color: #1a1714;
}

.wot-content {
  margin-bottom: 20px;
}
</style>
