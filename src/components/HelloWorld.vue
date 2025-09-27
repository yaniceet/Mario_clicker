<script setup>
import { ref, computed } from 'vue'
import { onMounted, onUnmounted, watch } from 'vue'

defineProps({
  msg: String,
})

const STORAGE_KEY_USERS = 'sonic_clicker_users'
const STORAGE_KEY_CURRENT_USER = 'sonic_clicker_current_user'
const STORAGE_KEY_CHALLENGES = 'sonic_clicker_challenges'

// État d'authentification
const currentUser = ref(null)
const isLoggedIn = ref(false)
const showAuthModal = ref(false)
const showProfileModal = ref(false)
const showLeaderboard = ref(false)
const showAdminPanel = ref(false)
const showChallenges = ref(false)
const username = ref('')
const users = ref([])
const challenges = ref([])
const notifications = ref([])

// État pour l'admin
const adminPassword = ref('')
const selectedUserForEdit = ref(null)
const newCookiesValue = ref(0)

// État du jeu
const cookies = ref(0)

const upgrades = ref([
  { id: 'assistant', name: 'Assistant', baseCost: 10, costGrowth: 1.15, qty: 0, baseRate: 0.1 },
  { id: 'usine', name: 'Usine', baseCost: 100, costGrowth: 1.15, qty: 0, baseRate: 2 },
])

const achievements = ref([
  { id: 'first_click', name: 'Premier clic', description: 'Cliquez pour la première fois', unlocked: false, condition: () => cookies.value >= 1 },
  { id: 'hundred_cookies', name: 'Centenaire', description: 'Récoltez 100 cookies', unlocked: false, condition: () => cookies.value >= 100 },
  { id: 'thousand_cookies', name: 'Millénaire', description: 'Récoltez 1 000 cookies', unlocked: false, condition: () => cookies.value >= 1000 },
  { id: 'ten_thousand_cookies', name: 'Dizaine de milliers', description: 'Récoltez 10 000 cookies', unlocked: false, condition: () => cookies.value >= 10000 },
  { id: 'first_assistant', name: 'Premier assistant', description: 'Achetez votre premier assistant', unlocked: false, condition: () => upgrades.value[0].qty >= 1 },
  { id: 'first_factory', name: 'Première usine', description: 'Achetez votre première usine', unlocked: false, condition: () => upgrades.value[1].qty >= 1 },
  { id: 'ten_assistants', name: 'Équipe d\'assistants', description: 'Achetez 10 assistants', unlocked: false, condition: () => upgrades.value[0].qty >= 10 },
  { id: 'production_machine', name: 'Machine de production', description: 'Ayez une production de 10/s', unlocked: false, condition: () => productionPerSecond.value >= 10 },
])

const getCost = (u) => Math.floor(u.baseCost * Math.pow(u.costGrowth, u.qty))

const productionPerSecond = computed(() => {
  return upgrades.value.reduce((acc, u) => acc + u.qty * u.baseRate, 0)
})

const unlockedAchievements = computed(() => {
  return achievements.value.filter(a => a.unlocked)
})

const lockedAchievements = computed(() => {
  return achievements.value.filter(a => !a.unlocked)
})

// Classement des joueurs
const leaderboard = computed(() => {
  return users.value
    .filter(u => u.role !== 'admin')
    .sort((a, b) => (b.cookies || 0) - (a.cookies || 0))
    .slice(0, 10)
})

// Vérifier si l'utilisateur est admin
const isAdmin = computed(() => {
  return currentUser.value && currentUser.value.role === 'admin'
})

// Défis actifs de l'utilisateur
const userChallenges = computed(() => {
  return challenges.value.filter(c => 
    c.challengerId === currentUser.value?.id || c.defenderId === currentUser.value?.id
  )
})

// Fonctions d'authentification
function loadUsers() {
  try {
    const saved = localStorage.getItem(STORAGE_KEY_USERS)
    if (saved) {
      users.value = JSON.parse(saved)
    }
  } catch (_) {
    users.value = []
  }
}

function loadChallenges() {
  try {
    const saved = localStorage.getItem(STORAGE_KEY_CHALLENGES)
    if (saved) {
      challenges.value = JSON.parse(saved)
    }
  } catch (_) {
    challenges.value = []
  }
}

function saveUsers() {
  try {
    localStorage.setItem(STORAGE_KEY_USERS, JSON.stringify(users.value))
  } catch (_) {}
}

function saveChallenges() {
  try {
    localStorage.setItem(STORAGE_KEY_CHALLENGES, JSON.stringify(challenges.value))
  } catch (_) {}
}

function login(usernameInput, password = '') {
  if (!usernameInput.trim()) return
  
  const user = users.value.find(u => u.username === usernameInput)
  if (user) {
    // Vérifier le mot de passe pour les admins
    if (user.role === 'admin' && password !== 'admin123') {
      addNotification('Mot de passe admin incorrect')
      return
    }
    
    currentUser.value = user
    isLoggedIn.value = true
    loadUserGameData()
    showAuthModal.value = false
    addNotification(`Bienvenue ${usernameInput}!`)
  } else {
    addNotification('Utilisateur introuvable')
  }
}

function register(usernameInput) {
  if (!usernameInput.trim()) return
  
  if (users.value.find(u => u.username === usernameInput)) {
    addNotification('Ce nom d\'utilisateur existe déjà')
    return
  }
  
  const newUser = {
    id: Date.now(),
    username: usernameInput,
    role: 'player',
    cookies: 0,
    upgrades: [
      { id: 'assistant', qty: 0 },
      { id: 'usine', qty: 0 }
    ],
    achievements: [
      { id: 'first_click', unlocked: false },
      { id: 'hundred_cookies', unlocked: false },
      { id: 'thousand_cookies', unlocked: false },
      { id: 'ten_thousand_cookies', unlocked: false },
      { id: 'first_assistant', unlocked: false },
      { id: 'first_factory', unlocked: false },
      { id: 'ten_assistants', unlocked: false },
      { id: 'production_machine', unlocked: false }
    ],
    createdAt: new Date().toISOString()
  }
  
  users.value.push(newUser)
  saveUsers()
  currentUser.value = newUser
  isLoggedIn.value = true
  loadUserGameData()
  showAuthModal.value = false
  addNotification(`Compte créé pour ${usernameInput}!`)
}

function logout() {
  saveUserGameData()
  currentUser.value = null
  isLoggedIn.value = false
  showProfileModal.value = false
  addNotification('Déconnexion réussie')
}

function loadUserGameData() {
  if (!currentUser.value) return
  
  cookies.value = currentUser.value.cookies || 0
  
  // Restaurer les upgrades
  currentUser.value.upgrades.forEach(savedUpgrade => {
    const upgrade = upgrades.value.find(u => u.id === savedUpgrade.id)
    if (upgrade) {
      upgrade.qty = savedUpgrade.qty || 0
    }
  })
  
  // Restaurer les achievements
  currentUser.value.achievements.forEach(savedAchievement => {
    const achievement = achievements.value.find(a => a.id === savedAchievement.id)
    if (achievement) {
      achievement.unlocked = savedAchievement.unlocked || false
    }
  })
}

function saveUserGameData() {
  if (!currentUser.value) return
  
  // Trouver l'utilisateur dans la liste et le mettre à jour
  const userIndex = users.value.findIndex(u => u.id === currentUser.value.id)
  if (userIndex !== -1) {
    users.value[userIndex].cookies = cookies.value
    users.value[userIndex].upgrades = upgrades.value.map(u => ({ id: u.id, qty: u.qty }))
    users.value[userIndex].achievements = achievements.value.map(a => ({ id: a.id, unlocked: a.unlocked }))
    currentUser.value = users.value[userIndex]
    saveUsers()
  }
}

// Fonctions d'administration
function createAdminUser() {
  if (!users.value.find(u => u.role === 'admin')) {
    const adminUser = {
      id: Date.now(),
      username: 'admin',
      role: 'admin',
      cookies: 0,
      upgrades: [],
      achievements: [],
      createdAt: new Date().toISOString()
    }
    users.value.push(adminUser)
    saveUsers()
    addNotification('Compte admin créé (username: admin, password: admin123)')
  }
}

function modifyUserScore(userId, newCookies) {
  const userIndex = users.value.findIndex(u => u.id === userId)
  if (userIndex !== -1) {
    users.value[userIndex].cookies = newCookies
    saveUsers()
    addNotification(`Score de ${users.value[userIndex].username} modifié: ${newCookies} cookies`)
  }
}

function resetUserGame(userId) {
  const userIndex = users.value.findIndex(u => u.id === userId)
  if (userIndex !== -1) {
    users.value[userIndex].cookies = 0
    users.value[userIndex].upgrades = [
      { id: 'assistant', qty: 0 },
      { id: 'usine', qty: 0 }
    ]
    users.value[userIndex].achievements = users.value[userIndex].achievements.map(a => ({
      ...a,
      unlocked: false
    }))
    saveUsers()
    addNotification(`Partie de ${users.value[userIndex].username} réinitialisée`)
  }
}

// Fonctions de défis
function challengePlayer(defenderId) {
  if (!currentUser.value || currentUser.value.id === defenderId) return
  
  const existingChallenge = challenges.value.find(c => 
    c.challengerId === currentUser.value.id && 
    c.defenderId === defenderId && 
    c.status === 'pending'
  )
  
  if (existingChallenge) {
    addNotification('Vous avez déjà un défi en attente avec ce joueur')
    return
  }
  
  const challenge = {
    id: Date.now(),
    challengerId: currentUser.value.id,
    challengerName: currentUser.value.username,
    defenderId: defenderId,
    defenderName: users.value.find(u => u.id === defenderId)?.username,
    status: 'pending',
    createdAt: new Date().toISOString(),
    challengerScore: currentUser.value.cookies,
    defenderScore: 0
  }
  
  challenges.value.push(challenge)
  saveChallenges()
  addNotification(`Défi envoyé à ${challenge.defenderName}!`)
}

function acceptChallenge(challengeId) {
  const challenge = challenges.value.find(c => c.id === challengeId)
  if (challenge) {
    challenge.status = 'active'
    challenge.defenderScore = currentUser.value.cookies
    saveChallenges()
    addNotification(`Défi accepté avec ${challenge.challengerName}!`)
  }
}

function declineChallenge(challengeId) {
  const challengeIndex = challenges.value.findIndex(c => c.id === challengeId)
  if (challengeIndex !== -1) {
    challenges.value.splice(challengeIndex, 1)
    saveChallenges()
    addNotification('Défi refusé')
  }
}

function addNotification(text) {
  const notification = {
    id: Date.now(),
    text,
    timestamp: Date.now()
  }
  notifications.value.push(notification)
  
  // Auto-remove after 3 seconds
  setTimeout(() => {
    const index = notifications.value.findIndex(n => n.id === notification.id)
    if (index > -1) {
      notifications.value.splice(index, 1)
    }
  }, 3000)
}

function checkAchievements() {
  achievements.value.forEach(achievement => {
    if (!achievement.unlocked && achievement.condition()) {
      achievement.unlocked = true
      addNotification(`🎉 Succès débloqué: ${achievement.name}!`)
    }
  })
}

function buyUpgrade(index) {
  const u = upgrades.value[index]
  const cost = getCost(u)
  if (cookies.value >= cost) {
    cookies.value -= cost
    u.qty += 1
    checkAchievements()
    saveUserGameData()
  }
}

let tickTimer = null

onMounted(() => {
  loadUsers()
  loadChallenges()
  createAdminUser()
  
  // Vérifier s'il y a un utilisateur connecté
  try {
    const savedCurrentUser = localStorage.getItem(STORAGE_KEY_CURRENT_USER)
    if (savedCurrentUser) {
      const userData = JSON.parse(savedCurrentUser)
      const user = users.value.find(u => u.id === userData.id)
      if (user) {
        currentUser.value = user
        isLoggedIn.value = true
        loadUserGameData()
      }
    }
  } catch (_) {}

  // start tick (1s)
  tickTimer = setInterval(() => {
    if (productionPerSecond.value > 0 && isLoggedIn.value) {
      cookies.value += productionPerSecond.value
    }
  }, 1000)
})

onUnmounted(() => {
  if (tickTimer) clearInterval(tickTimer)
})

watch(cookies, (value) => {
  if (isLoggedIn.value) {
    saveUserGameData()
    checkAchievements()
  }
})

watch(upgrades, (value) => {
  if (isLoggedIn.value) {
    saveUserGameData()
  }
}, { deep: true })

watch(achievements, (value) => {
  if (isLoggedIn.value) {
    saveUserGameData()
  }
}, { deep: true })

watch(currentUser, (value) => {
  try {
    if (value) {
      localStorage.setItem(STORAGE_KEY_CURRENT_USER, JSON.stringify(value))
    } else {
      localStorage.removeItem(STORAGE_KEY_CURRENT_USER)
    }
  } catch (_) {}
}, { deep: true })
</script>

<template>
  <h1>{{ msg }}</h1>

  <!-- Notifications -->
  <div class="notifications">
    <div 
      v-for="notification in notifications" 
      :key="notification.id"
      class="notification"
    >
      {{ notification.text }}
    </div>
  </div>

  <!-- Authentification -->
  <div v-if="!isLoggedIn" class="auth-section">
    <div class="card">
      <h2>Connexion / Inscription</h2>
      <div class="auth-form">
        <input 
          v-model="username" 
          type="text" 
          placeholder="Nom d'utilisateur"
          @keyup.enter="login(username)"
        />
        <div class="auth-buttons">
          <button @click="login(username)">Se connecter</button>
          <button @click="register(username)">S'inscrire</button>
        </div>
      </div>
      
      <div v-if="users.length > 0" class="existing-users">
        <h3>Profils existants</h3>
        <ul>
          <li v-for="user in users" :key="user.id" class="user-item">
            <span>{{ user.username }} - {{ Math.floor(user.cookies) }} cookies</span>
            <button @click="login(user.username)">Charger</button>
          </li>
        </ul>
      </div>
    </div>
  </div>

  <!-- Interface de jeu -->
  <div v-else class="game-section">
    <!-- Barre utilisateur -->
    <div class="user-bar">
      <span>Joueur: <strong>{{ currentUser.username }}</strong> 
        <span v-if="isAdmin" class="admin-badge">👑 Admin</span>
      </span>
      <div class="user-actions">
        <button @click="showProfileModal = true">Profil</button>
        <button @click="showLeaderboard = true">Classement</button>
        <button @click="showChallenges = true">Défis</button>
        <button v-if="isAdmin" @click="showAdminPanel = true">Admin</button>
        <button @click="logout()">Déconnexion</button>
      </div>
    </div>

    <div class="card">
      <div class="mario-clicker">
        <h3>🍄 Cliquer sur le champignon pour renforcer Mario</h3>
        <button type="button" @click="cookies++; checkAchievements()" class="mushroom-button">
          <div class="mario-mushroom">
            <div class="mushroom-cap"></div>
            <div class="mushroom-spots"></div>
            <div class="mushroom-stem"></div>
          </div>
        </button>
        <p class="cookie-count">{{ Math.floor(cookies) }}</p>
        <p>Production: {{ productionPerSecond.toFixed(2) }} / sec</p>
      </div>
    </div>

  <div class="card">
    <h2>Améliorations</h2>
    <ul>
      <li v-for="(u, idx) in upgrades" :key="u.id" style="margin: 0.5rem 0;">
        <strong>{{ u.name }}</strong>
        — possédé: {{ u.qty }}
        — coût: {{ getCost(u) }} cookies
        — +{{ u.baseRate }}/s chacun
        <button
          type="button"
          @click="buyUpgrade(idx)"
          :disabled="cookies < getCost(u)"
          style="margin-left: 0.5rem;"
        >Acheter</button>
      </li>
    </ul>
  </div>

    <div class="card">
      <h2>Succès ({{ unlockedAchievements.length }}/{{ achievements.length }})</h2>
      
      <div v-if="unlockedAchievements.length > 0">
        <h3>🎉 Débloqués</h3>
        <ul>
          <li v-for="achievement in unlockedAchievements" :key="achievement.id" class="achievement-unlocked">
            <strong>{{ achievement.name }}</strong> - {{ achievement.description }}
          </li>
        </ul>
      </div>
      
      <div v-if="lockedAchievements.length > 0">
        <h3>🔒 Verrouillés</h3>
        <ul>
          <li v-for="achievement in lockedAchievements" :key="achievement.id" class="achievement-locked">
            <strong>{{ achievement.name }}</strong> - {{ achievement.description }}
          </li>
        </ul>
      </div>
    </div>
  </div>

  <!-- Modal de profil -->
  <div v-if="showProfileModal" class="modal-overlay" @click="showProfileModal = false">
    <div class="modal" @click.stop>
      <h2>Profil de {{ currentUser.username }}</h2>
      <div class="profile-info">
        <p><strong>Cookies:</strong> {{ Math.floor(currentUser.cookies) }}</p>
        <p><strong>Production:</strong> {{ productionPerSecond.toFixed(2) }}/s</p>
        <p><strong>Succès débloqués:</strong> {{ unlockedAchievements.length }}/{{ achievements.length }}</p>
        <p><strong>Compte créé:</strong> {{ new Date(currentUser.createdAt).toLocaleDateString() }}</p>
      </div>
      <button @click="showProfileModal = false">Fermer</button>
    </div>
  </div>

  <!-- Modal classement -->
  <div v-if="showLeaderboard" class="modal-overlay" @click="showLeaderboard = false">
    <div class="modal leaderboard-modal" @click.stop>
      <h2>🏆 Classement des joueurs</h2>
      <div class="leaderboard-list">
        <div v-for="(user, index) in leaderboard" :key="user.id" class="leaderboard-item">
          <span class="rank">#{{ index + 1 }}</span>
          <span class="username">{{ user.username }}</span>
          <span class="score">{{ Math.floor(user.cookies) }} cookies</span>
          <button 
            v-if="!isAdmin && user.id !== currentUser.id" 
            @click="challengePlayer(user.id)"
            class="challenge-btn"
          >
            Défier
          </button>
        </div>
      </div>
      <button @click="showLeaderboard = false">Fermer</button>
    </div>
  </div>

  <!-- Modal défis -->
  <div v-if="showChallenges" class="modal-overlay" @click="showChallenges = false">
    <div class="modal challenges-modal" @click.stop>
      <h2>⚔️ Défis</h2>
      
      <div v-if="userChallenges.length === 0" class="no-challenges">
        <p>Aucun défi actif</p>
      </div>
      
      <div v-else>
        <div v-for="challenge in userChallenges" :key="challenge.id" class="challenge-item">
          <div class="challenge-info">
            <p><strong>{{ challenge.challengerName }}</strong> vs <strong>{{ challenge.defenderName }}</strong></p>
            <p>Status: {{ challenge.status === 'pending' ? 'En attente' : 'Actif' }}</p>
            <p>Créé: {{ new Date(challenge.createdAt).toLocaleDateString() }}</p>
          </div>
          
          <div v-if="challenge.status === 'pending' && challenge.defenderId === currentUser.id" class="challenge-actions">
            <button @click="acceptChallenge(challenge.id)" class="accept-btn">Accepter</button>
            <button @click="declineChallenge(challenge.id)" class="decline-btn">Refuser</button>
          </div>
        </div>
      </div>
      
      <button @click="showChallenges = false">Fermer</button>
    </div>
  </div>

  <!-- Modal admin -->
  <div v-if="showAdminPanel && isAdmin" class="modal-overlay" @click="showAdminPanel = false">
    <div class="modal admin-modal" @click.stop>
      <h2>👑 Panel d'administration</h2>
      
      <div class="admin-section">
        <h3>Gestion des utilisateurs</h3>
        <div v-for="user in users.filter(u => u.role !== 'admin')" :key="user.id" class="admin-user-item">
          <div class="user-info">
            <span><strong>{{ user.username }}</strong> - {{ Math.floor(user.cookies) }} cookies</span>
          </div>
          <div class="admin-actions">
            <input 
              v-model.number="newCookiesValue" 
              type="number" 
              placeholder="Nouveau score"
              :id="`score-${user.id}`"
            />
            <button @click="modifyUserScore(user.id, newCookiesValue)">Modifier score</button>
            <button @click="resetUserGame(user.id)" class="reset-btn">Réinitialiser</button>
          </div>
        </div>
      </div>
      
      <button @click="showAdminPanel = false">Fermer</button>
    </div>
  </div>

  <p>
    Check out
    <a href="https://vuejs.org/guide/quick-start.html#local" target="_blank"
      >create-vue</a
    >, the official Vue + Vite starter
  </p>
  <p>
    Learn more about IDE Support for Vue in the
    <a
      href="https://vuejs.org/guide/scaling-up/tooling.html#ide-support"
      target="_blank"
      >Vue Docs Scaling up Guide</a
    >.
  </p>
  <p class="read-the-docs">Click on the Vite and Vue logos to learn more</p>
</template>

<style scoped>
.read-the-docs {
  color: rgba(255, 255, 255, 0.8);
}

.notifications {
  position: fixed;
  top: 20px;
  right: 20px;
  z-index: 1000;
}

.notification {
  background: linear-gradient(135deg, #FF6B35 0%, #F7931E 100%);
  color: white;
  padding: 12px 16px;
  margin-bottom: 8px;
  border-radius: 15px;
  border: 2px solid #FFD700;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
  animation: slideIn 0.3s ease-out;
  max-width: 300px;
  font-weight: bold;
}

@keyframes slideIn {
  from {
    transform: translateX(100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

.achievement-unlocked {
  color: #FFD700;
  font-weight: bold;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
}

.achievement-locked {
  color: rgba(255, 255, 255, 0.6);
  opacity: 0.7;
}

.auth-section {
  max-width: 600px;
  margin: 0 auto;
}

.auth-form {
  margin: 1rem 0;
}

.auth-form input {
  width: 100%;
  padding: 0.5rem;
  margin-bottom: 1rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1rem;
}

.auth-buttons {
  display: flex;
  gap: 1rem;
}

.auth-buttons button {
  flex: 1;
  padding: 0.5rem 1rem;
  background: linear-gradient(135deg, #FF6B35 0%, #F7931E 100%);
  color: white;
  border: 2px solid #FFD700;
  border-radius: 15px;
  cursor: pointer;
  font-weight: bold;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
}

.auth-buttons button:hover {
  background: linear-gradient(135deg, #FF8E53 0%, #FFB347 100%);
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
}

.existing-users {
  margin-top: 2rem;
  padding-top: 2rem;
  border-top: 1px solid #eee;
}

.user-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem 0;
  border-bottom: 1px solid #f0f0f0;
}

.user-item button {
  padding: 0.25rem 0.75rem;
  background: #667eea;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.9rem;
}

.user-item button:hover {
  background: #5a6fd8;
}

.user-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 15px;
  margin-bottom: 1rem;
  border: 2px solid rgba(255, 215, 0, 0.3);
  backdrop-filter: blur(10px);
}

.user-bar button {
  padding: 0.5rem 1rem;
  margin-left: 0.5rem;
  border: 2px solid #FFD700;
  border-radius: 10px;
  background: linear-gradient(135deg, #FF6B35 0%, #F7931E 100%);
  color: white;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.3s ease;
}

.user-bar button:hover {
  background: linear-gradient(135deg, #FF8E53 0%, #FFB347 100%);
  transform: translateY(-2px);
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal {
  background: linear-gradient(135deg, rgba(76, 175, 80, 0.95) 0%, rgba(139, 195, 74, 0.95) 50%, rgba(205, 220, 57, 0.95) 100%);
  padding: 2rem;
  border-radius: 20px;
  border: 3px solid #FFD700;
  max-width: 400px;
  width: 90%;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(10px);
  color: #fff;
}

.profile-info {
  margin: 1rem 0;
}

.profile-info p {
  margin: 0.5rem 0;
  color: #fff;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
}

.admin-badge {
  background: linear-gradient(135deg, #ff6b6b, #ee5a24);
  color: white;
  padding: 0.2rem 0.5rem;
  border-radius: 12px;
  font-size: 0.8rem;
  margin-left: 0.5rem;
}

.user-actions {
  display: flex;
  gap: 0.5rem;
}

.leaderboard-modal {
  max-width: 600px;
  background: linear-gradient(135deg, rgba(76, 175, 80, 0.95) 0%, rgba(139, 195, 74, 0.95) 50%, rgba(205, 220, 57, 0.95) 100%);
  color: #fff;
}

.leaderboard-list {
  max-height: 400px;
  overflow-y: auto;
  margin: 1rem 0;
}

.leaderboard-item {
  display: flex;
  align-items: center;
  padding: 0.75rem;
  border-bottom: 1px solid #eee;
  gap: 1rem;
}

.rank {
  font-weight: bold;
  color: #667eea;
  min-width: 2rem;
}

.username {
  flex: 1;
  font-weight: 500;
}

.score {
  color: #666;
  min-width: 6rem;
  text-align: right;
}

.challenge-btn {
  background: #ff6b6b;
  color: white;
  border: none;
  padding: 0.25rem 0.75rem;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.9rem;
}

.challenge-btn:hover {
  background: #ff5252;
}

.challenges-modal {
  max-width: 500px;
  background: linear-gradient(135deg, rgba(76, 175, 80, 0.95) 0%, rgba(139, 195, 74, 0.95) 50%, rgba(205, 220, 57, 0.95) 100%);
  color: #fff;
}

.no-challenges {
  text-align: center;
  padding: 2rem;
  color: rgba(255, 255, 255, 0.8);
}

.challenge-item {
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 1rem;
  background: rgba(255, 255, 255, 0.1);
}

.challenge-info p {
  margin: 0.25rem 0;
  color: #fff;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
}

.challenge-actions {
  margin-top: 1rem;
  display: flex;
  gap: 0.5rem;
}

.accept-btn {
  background: #4caf50;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}

.decline-btn {
  background: #f44336;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}

.admin-modal {
  max-width: 700px;
  background: linear-gradient(135deg, rgba(76, 175, 80, 0.95) 0%, rgba(139, 195, 74, 0.95) 50%, rgba(205, 220, 57, 0.95) 100%);
  color: #fff;
}

.admin-section {
  margin: 1rem 0;
}

.admin-user-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 8px;
  margin-bottom: 1rem;
  background: rgba(255, 255, 255, 0.1);
}

.user-info {
  flex: 1;
  color: #fff;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
}

.admin-actions {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}

.admin-actions input {
  width: 120px;
  padding: 0.25rem;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.admin-actions button {
  padding: 0.25rem 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  cursor: pointer;
  font-size: 0.9rem;
}

.reset-btn {
  background: #f44336 !important;
  color: white !important;
  border: none !important;
}

.reset-btn:hover {
  background: #d32f2f !important;
}

/* Styles pour le bouton champignon */
.mario-clicker {
  text-align: center;
  padding: 2rem;
}

.mario-clicker h3 {
  color: #fff;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
  margin-bottom: 2rem;
  font-size: 1.5rem;
}

.mushroom-button {
  width: 140px;
  height: 140px;
  border: none;
  background: transparent;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 1rem auto;
  padding: 0;
}

.mushroom-button:hover {
  transform: scale(1.1);
}

.mushroom-button:hover .mario-mushroom {
  filter: brightness(1.2);
}

.mushroom-button:active {
  transform: scale(0.95);
}

.mushroom-button:active .mario-mushroom {
  animation: mushroom-bounce 0.3s ease;
}

.cookie-count {
  font-size: 1.5rem;
  font-weight: bold;
  color: #FFD700;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
  margin: 1rem 0;
}

@keyframes pulse {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.05);
  }
  100% {
    transform: scale(1);
  }
}

@keyframes mushroom-bounce {
  0% {
    transform: scale(0.95);
  }
  50% {
    transform: scale(1.15);
  }
  100% {
    transform: scale(1);
  }
}

/* Styles pour le champignon Mario authentique */
.mario-mushroom {
  position: relative;
  width: 120px;
  height: 100px;
  animation: pulse 2s infinite;
}

.mushroom-cap {
  position: absolute;
  top: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 100px;
  height: 70px;
  background: linear-gradient(135deg, #FF0000 0%, #CC0000 50%, #AA0000 100%);
  border-radius: 50px 50px 35px 35px;
  border: 3px solid #8B0000;
  box-shadow: 
    inset 0 5px 10px rgba(255, 255, 255, 0.3),
    inset 0 -5px 10px rgba(0, 0, 0, 0.3),
    0 3px 8px rgba(0, 0, 0, 0.4);
}

.mushroom-spots {
  position: absolute;
  top: 15px;
  left: 50%;
  transform: translateX(-50%);
  width: 90px;
  height: 55px;
}

.mushroom-stem {
  position: absolute;
  bottom: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 35px;
  height: 40px;
  background: linear-gradient(135deg, #F5DEB3 0%, #DEB887 50%, #D2B48C 100%);
  border: 3px solid #CD853F;
  border-radius: 50% 50% 0 0;
  box-shadow: 
    inset 2px 0 5px rgba(0, 0, 0, 0.1),
    inset -2px 0 5px rgba(255, 255, 255, 0.5),
    0 2px 5px rgba(0, 0, 0, 0.2);
}

/* Ajouter une tache supplémentaire */
.mario-mushroom::before {
  content: '';
  position: absolute;
  top: 30px;
  left: 50%;
  transform: translateX(-50%);
  width: 10px;
  height: 10px;
  background: #FFFFFF;
  border-radius: 50%;
  border: 1px solid #F0F0F0;
}

/* Ajouter un rond blanc au centre du chapeau */
.mushroom-cap::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 35px;
  height: 35px;
  background: #FFFFFF;
  border-radius: 50%;
  border: 3px solid #F0F0F0;
  box-shadow: 0 3px 6px rgba(0, 0, 0, 0.15);
}

/* Ajouter les yeux sur la tige */
.mushroom-stem::before,
.mushroom-stem::after {
  content: '';
  position: absolute;
  background: #000000;
  border-radius: 50%;
}

.mushroom-stem::before {
  top: 12px;
  left: 8px;
  width: 6px;
  height: 8px;
}

.mushroom-stem::after {
  top: 12px;
  right: 8px;
  width: 6px;
  height: 8px;
}
</style>
