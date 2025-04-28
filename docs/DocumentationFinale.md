# TP2 Revue de code.

Ceci est la revue de code du travail pratique 2.

## AccueilView.vue

- Page d'accueil AccueilView.vue avec titre "Credit Rush".
- Ajout des routes pour pouvoir revenir à la page d'accueil:

  ~~~~md
  const routes: Array<RouteRecordRaw> = [
      {
        // Accueil
        path: '/',
        name: 'Accueil',
        component: AccueilView
      },
  ~~~~
- Formulaire pour le nom et le vaisseau validé par une méthode validateForm.

  ~~~~md
  const validateForm = computed(() => {
      return !playerName.value.trim()
          || !shipName.value.trim()
  })
  ~~~~

- Création du joueur une fois le formulaire validé.

  ~~~~md
    export function initPlayer(playerName: string, shipName: string) {
        const newPlayer: Player = {
            name: playerName.trim(),
            xp: "Maitre",
            cgWon: 0,
            shipName: shipName,
            vitality: 100
        };

        return newPlayer
    }
  ~~~~
- Utilisation d'interface Player et de Ship.

  ~~~~md
  import type { Ship } from '@/scripts/ship.ts';

  export interface Player {
    name: string;
    xp: string;
    cgWon: number;
    shipName: string;
    vitality: number;
  }

  ============

  export interface Ship {
      id: number
      name: string
  }
  ~~~~

* Méthode de transfert du player changé pour s'adapter à la classe GameView, utilisation d'un champ query pour passer les informations sous forme de string:

  ~~~md
  import { shipService } from '../services/ShipService.ts'

  const router = useRouter()

  const playerName = ref<string>('')
  const shipName = ref<string>('')

  const ships = ref<Ship[]>([])

  onMounted(async () => {
      ships.value = await shipService.getShips()
  })

  function initPlayer() {
      const newPlayer: Player = {
          name: playerName.value.trim(),
          xp: "Maitre",
          cgWon: 0,
          shipName: shipName.value,
          vitality: 100
      };

      return newPlayer
  }

  // router.push() avec query trouvé ici:
  // https://router.vuejs.org/guide/essentials/navigation.html#accessing-query-params
  function startGame() {
      const player = initPlayer()
      router.push({ name: 'Game',
          query: {
              name: player.name,
              xp: player.xp,
              cgWon: player.cgWon.toString(),
              vitality: player.vitality.toString(),
              shipName: player.shipName
          } 
      })
  }

  ~~~
* RouterLink changé par un bouton dans le template qui appelle la méthode startGame() qui nous redirige sur le GameView:

~~~md
<button class="btn btn-success" :disabled="validateForm" @click="startGame">
Lancer la partie
</button>
~~~

- Utilisation du service ShipService.ts avec deux méthodes, getShips() qui retourne tous les vaisseaux et getShip() qui retourne un vaisseau recherché par id.

  ~~~~md
  import axios from 'axios'

  const API_URL = 'http://localhost:3000/ships'

  async function getShips () {
      const { data } = await axios.get(`${API_URL}`)
      return data
  }

  async function getShip (id : string) {
      const { data } = await axios.get(`${API_URL}/${id}`)
      return data
  }

  export const shipService = {
      getShips,
      getShip
  }
  ~~~~
- 4 tests unitaires pour la vue Accueil; Un vérifie qu'on ne peut pas lancer de partie si aucun champ du formulaire n'est rempli, deux vérifie que si l'un ou l'autre des champs et rempli sans l'autre on ne peut pas lancer de partie et un vérifie que si le formulaire est valide on peut lancer une partie.

## ScoreView.vue

**Obtention tableau de classement**

Le triage se fait en temps réel, ce qui permet au classement d'être calibré en tout temps.

````md
data(){
    return{
        myJson: json.ranking.sort((a,b) => b.score - a.score)
    }
}
````

**Indexage**

J'ai utilisé .indexOf à la place du ID pour faire les rangs, car l'ID doit
être utilisé pour identifier le joueur (Comme une clé unique) et sa position
dans le classement fait déjà le travail pour indexer.

````md
<td>{{ myJson.indexOf(ranking) + 1 }}</td>
````

## GameView.vue

* La classe est divisée en deux grosses parties, le player et l'ennemi.
* Récupération des informations du query passées en paramètre dans le router.push() dans le onMounted du GameView en s'assurant qu'ils sont sous forme de string:

~~~md
onMounted(async () => {
    enemies.value = await enemyService.getEnemies()

    ships.value = await shipService.getShips()
    ship.value = ships.value.find(ship => ship.name === route.query.shipName)

    if (
        typeof route.query.name === 'string' &&
        typeof route.query.xp === 'string' &&
        typeof route.query.cgWon === 'string' &&
        ship.value &&
        typeof route.query.vitality === 'string'
    ) {
        player.value = {
            name: route.query.name,
            xp: route.query.xp,
            cgWon: parseInt(route.query.cgWon),
            vitality: parseInt(route.query.vitality),
            shipName: ship.value.name
        }
    }
})
~~~

La barre de santé utilise des components de Vue.js (comme :style) pour permettre une largeur et une valeur maximale qui s'adapte en temps réelle et/ou pour pouvoir utiliser des constantes.

````md
<div
    class="progress-bar"
    :class="{
    'bg-success': player.vitality >= (MAX_PLAYER_HP / 2),
    'bg-warning text-dark': player.vitality < (MAX_PLAYER_HP / 2) && player.vitality >= (MAX_PLAYER_HP / 4),
    'bg-danger': player.vitality < (MAX_PLAYER_HP / 4)
     }"
     role="progressbar"
     :style="{width: player.vitality + '%'}"
     aria-valuemin="0"
     :aria-valuemax="MAX_PLAYER_HP"
     >
     {{ player.vitality }} % ({{ player.vitality + "/" + MAX_PLAYER_HP}})
</div>
````
* Gestion du niveau et de la précision de l'ennemi dans une méthode retournant la bonne valeur dépendamment du niveau en question au travers de switch case ex:

~~~md
function ennemyLevel(xp : number) {
    switch (xp) {
        case 1:
            return "débutant";
        case 2:
            return "confirmé";
        case 3:
            return "expert";
        case 4:
            return "Maitre";
        default:
            break;
    }
    return 0
}
~~~

* Pour chercher un ennemi aléatoire dans la base de donnée, utilisation d'un EnnemyService dans /services construit exactement comme le ShipService:

~~~md
const currentEnemy = ref<Enemy>(enemies.value[0])
const MAX_ENEMY_HP = ref<number>(0)

function selectEnemy() {
    const enemy = getRandomEnemy(enemies.value)
    currentEnemy.value = enemy
    MAX_ENEMY_HP.value = enemy.ship.vitality
}

export function getRandomEnemy(enemies: Enemy[]) {
    const i = Math.floor(Math.random() * enemies.length);
    return enemies[i]
}
~~~

* Ajout de la méthode attack() avec le bouton attaquer pour combattre l'ennemi. Un ennemi attaque tout de suite après le joueur et si l'un deux meurt l'évènement approprié se produit.

~~~md
const lowerDamage = ref<number>(3)
const higherDamage = ref<number>(6)

function attack() {
    if(validateDamageInput(higherDamage.value, lowerDamage.value)){
        return
    }

    if (player.value && currentEnemy.value){

        const enemyHitChance = enemyPrecision(currentEnemy.value.experience)
        const playerHitChance = 70

        const playerHit = Math.random() * 100 < playerHitChance
        const enemyHit = Math.random() * 100 < enemyHitChance

        const damageToTake = () => {
            const min = lowerDamage.value
            const max = higherDamage.value
            return Math.floor(Math.random() * (max - min + 1)) + min
        }

        // Player attack
        if (playerHit) {
            currentEnemy.value.ship.vitality -= damageToTake()
            if (currentEnemy.value.ship.vitality <= 0) {
                currentEnemy.value.ship.vitality = 0
                player.value.cgWon += currentEnemy.value.credit
                emit('wonModal')
                skipMission()
                //Besoin du return pour arrêter la fonction et empêcher l'ennemi d'attaquer même après sa mort.
                return
            }
        }

        // Enemy attack
        if (enemyHit) {
            player.value.vitality -= damageToTake()
            if (player.value.vitality <= 0) {
                emit('lostModal')
            }
        }
    }
}

TEMPLATE

<button type="button" class="btn btn-danger mt-2" @click="attack()">
    Attaquer l'ennemi
</button>
~~~

* Méthode de validation pour s'assurer que les dégâts maximal soit plus grand que les dégâts minimal.

~~~md
export function validateDamageInput(higherDamage:number, lowerDamage:number) {
    if (higherDamage < lowerDamage) {
        alert("Les dégâts max doivent être plus grands que les dégâts min")
        return true
    }
    return false
}
~~~

* Choix de la difficulté fait dans une composante DifficultyComponent.vue

~~~md
Dégât minimal (%):
<select :value="props.lowerDamage"
        @change="event => emit('update:lowerDamage', Number((event.target as HTMLSelectElement).value))"
        class="select">
    <option disabled value="3">3</option>
    <option v-for="nb in 15" :key="'min'+nb" :value="nb">
            {{ String(nb) }}
    </option>
</select>
Dégât maximal (%):
<select :value="props.higherDamage"
        @change="event => emit('update:higherDamage', Number((event.target as HTMLSelectElement).value))"
        class="select">
    <option disabled value="6">6</option>
    <option v-for="nb in 20" :key="'max'+nb" :value="nb">
            {{ String(nb) }}
    </option>
</select>
~~~

Ajout d'un watch pour savoir si le joueur est en vie ou non.
S'il ne l'est pas, un modal est affiché à l'écran.

```md
watch(() => player.value?.vitality,  (newVitality) => {
    nextTick()
    if(newVitality == undefined || newVitality <= 0){
        isAlive.value = false
        emit('lostModal');
        triggerLostModal.value++;
    }
    return isAlive.value;
})
```

Fonction qui permet de passer d'une mission à une autre et si le score est > 0, alors le joueur vera son score et son nom dans le tableau de classement.

```md
if(currentMission.value > AMOUNT_OF_MISSIONS && player.value.cgWon > 0){
    rankingService.setRanking(player.value?.name, player.value.cgWon.toString())
    router.push('/score')
}
```

Fonction qui permet de réparer son vaisseau MAIS rester dans le combat actuel.
J'ai pris comme décision d'implémenter cette fonctionnalité car je trouve qu'en tant que joueur,
je devrais être capable de réparer mon vaisseau tout en restant en combat pour pouvoir vaincre
mon ennemi et récupérer ses crédits.

```md
const hpRegenerate = basePlayerHp / 100 // Pour avoir le pourcentage

while(player.cgWon >= cgCostPerHp && player.vitality < basePlayerHp){
    player.cgWon -= cgCostPerHp
    player.vitality += hpRegenerate
}
```

Fonction qui permet de passer d'une mission à une autre ET de réparer son vaisseau.

```md
function manipulateTime(player: Player){
    repairShip(player, MAX_PLAYER_HP, CG_COST_PER_HP)
    skipMission()
}
```

* Un LostModal et un WonModal ont été ajoutés pour bénéficier l'expérience du joueur.

* Les fonctions propre à certains objets comme Player ou Enemy ont été déplacées dans leur fichier script respectif et les éléments HTML volumineux ont été mis dans des composantes à part.

* 9 tests unitaires ajouté. Les cas testés sont:

    - Devrait avoir un ennemi au début de la partie
    - Après skipMission() un nouvel ennemi doit apparaitre
    - Après skipMission() la mission est incrémenté
    - Devrait réparer le vaisseau
    - Ne devrait pas réparer le vaisseau si les crédits sont insuffisant
    - Devrait faire perdre de la vie ennemi après une attaque
    - Montre wonModal quand on bat un ennemi
    - Montre lostModal quand on meurt
    - Devrait push sur /score quand on a gagné

* Dans les tests, le wrapper est comme tel pour permettre de garder et tester les composantes nécessaire au GameView. L'utilisation de VueWrapper nous permets d'accéder à tout ce qui se trouve dans GameView sans soucis.

~~~md
const wrapper = mount(GameView, {
    global: { stubs: ['RouterLink', 'LostModal', 'WonModal'] }
}) as VueWrapper<any>
~~~

## LostModal.vue

Le modal utilisé quand le joueur a perdu est statique, ce qui veut dire que la seule façon qu'il puisse interagir avec la page est en cliquant sur le bouton d'accueil.

```md
<div class="modal fade" id="staticBackdrop" data-bs-backdrop="static" data-bs-keyboard="false" tabindex="-1" role="dialog" aria-labelledby="staticBackdropLabel">
...
```

Ce même bouton retourne le joueur à l'accueil, comme s'il n'avait pas encore joué.

```md
...
<button type="button" class="btn btn-primary" name="backHome" data-bs-dismiss="modal" @click="confirm()">
    <RouterLink class="nav-link" id="accueil" to="/">Retour à l'accueil</RouterLink>
</button>
...
```

## WonModal.vue

Ce modal-ci par contre n'est pas statique, tout ce qu'il fait c'est informer le joueur qu'il a tué tel ennemi et récolté tant de crédits.

```md
...
<div class="modal-header">
    <h1 class="modal-title fs-5" id="exampleModalLabel">Bien joué {{ playerName }}!</h1>
    <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
</div>
<div class="modal-body">
    Vous avez vaincu {{ enemyName }} ! Vous remportez {{ enemyCredit }} crédits galactiques !
</div>
...
```

## RankingService.ts

Cette fonction dans setRanking() permet de transférer le name du joueur et son score dans une base de données pour garder son score même s'il quitte le site.

```md
...
const jsonData = JSON.stringify({name, score }, null, 2);

await fetch(API_URL, {
    method: 'POST',
    headers: {
        'Content-Type' : 'application/json'
    },
    body: jsonData
})
```

## Tests unitaires

LostModal a été testé pour vérifier s'il affichait et s'il ramenait à l'accueil.
Pointage a été testé pour vérifier s'il pouvait accepter de nouveaux scores.
GameView a été testé pour vérifier si le déroulement normal du jeu se passe comme prévu.