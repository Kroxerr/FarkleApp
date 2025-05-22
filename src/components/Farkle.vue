<script setup>
    // --- PS: VŠE PÍŠU ANGLICKY, PROTOŽE JE TO PODLE MĚ LEPŠÍ --- //
    // --- 'var' už nepoužívám, podle kámoše se 'var' nepoužívá, nebo aspoň není důvod ho používat místo 'let' --- ///
    
    import { ref, onMounted } from 'vue'

    // Dice array to pick from yaaay //
    const dices = [
        { value: 1, img: "/FarkleApp/Farkle/1.png", score: 100 },
        { value: 2, img: "/FarkleApp/Farkle/2.png", score:   0 },
        { value: 3, img: "/FarkleApp/Farkle/3.png", score:   0 },
        { value: 4, img: "/FarkleApp/Farkle/4.png", score:   0 },
        { value: 5, img: "/FarkleApp/Farkle/5.png", score:  50 },
        { value: 6, img: "/FarkleApp/Farkle/6.png", score:   0 },
    ]

    //Note to self//
    // = nastavuje
    // == porovnává
    // === absolutní porovnání, i typ, jestli to je string apod.


    //---GAME - WHAT SCORE TO PLAY TO---//
    const scoreToPlay   = ref(1000)



    //All the stuff I have to track//
    const gameTitle     = ref('FARKLE') // can change text for the selected score
    const score         = ref(0)        // selected score in the current round
    const scoreBanked   = ref(0)        // banked score
    const result        = ref([])       // current roll of dice to play with
    const storedDice    = ref([])       // banked dice
    const gameState     = ref('start')  // start | firstRound | nthRound | gameOver
    const currentPlayer = ref('player1')// what player is currently playing
    const pOneScore     = ref(0)        // score of player1
    const pTwoScore     = ref(0)        // score of player2
    const warningState  = ref ('off')   // toggles warning message if the player does something against the rules
    const warningMsg    = ref('If you see this message, my app is broken somehow :D')// Input for what the warning message should say

    //Shuffle Dice + Start Game//
    function shuffleDice(nthRound) {
        result.value = [];

        // this triggers the first round of the game or the next round when banking dice
        if (gameState.value === 'start') {
            gameState.value = 'firstRound'; // mohl by to být nthRound gameState, ale nechal jsem ho tady od začátku a od té doby jsem líný to změnit :DD
            gameTitle.value = 'SELECTED SCORE'
            console.log('GameState is: PLAYING');
        } else {
            gameState.value = 'nthRound';
        }

        // shuffles dice
        if (nthRound && storedDice.value.length === 6) {    // if banked dice = 6 then it resets them to none and shuffles all 6 dice
            for (let i = 0; i < 6; i++) {
                let randomNumber = Math.floor(Math.random() * dices.length) + 1;    //ChatGPT helped with math
                let randomDice = dices.find(item => item.value == randomNumber);
                result.value.push({ ...randomDice, isSelected: false });    // isSelected: false - just to make sure no dice remains selected
            }
            storedDice.value = [];
        } else {
            for (let i = 0; i < 6 - storedDice.value.length; i++) {     // rolls dice that is not in the bank
                let randomNumber = Math.floor(Math.random() * dices.length) + 1; //ChatGPT helped with math
                let randomDice = dices.find(item => item.value == randomNumber);
                result.value.push({ ...randomDice, isSelected: false });    // isSelected: false - just to make sure no dice remains selected
            }
        }

        // === FARKLE CHECK === //
        // ChatGPT napsal většinu funkce co kontroluje jestli ve hře je nějaké kombo - pokud není tak kolo skončilo

        const tempCount = {}; // counts how many times each dice value (1 to 6) appears in the game board to play with

        result.value.forEach(dice => {
            tempCount[dice.value] = (tempCount[dice.value] || 0) + 1;   // dice.value = is the number on the dice
        });                                                             // check dice and what value it is and adds a 1 to the total count of that dice

        const specialScore = detectSpecialCombos(tempCount);
        let hasScoringCombo = specialScore > 0;

        if (!hasScoringCombo) {
            for (let val in tempCount) {
                const count = tempCount[val];
                const number = parseInt(val);   //converts the key (which is a string) into a number

                // if dice 1 or dice 5 is there at least once - the game continues
                if ((number === 1 || number === 5) && count >= 1) {
                    hasScoringCombo = true;
                    break;
                }

                // if any dice value appears 3 or more times - game continues
                if (count >= 3) {
                    hasScoringCombo = true;
                    break;
                }
            }
        }

        // If there is no dice combo that has points value - end the game - simple as that
        if (!hasScoringCombo) {
            gameState.value = 'FARKLE';
            warningState.value = 'on';
            warningMsg.value = 'You lost the round';
            scoreBanked.value = 0;
        } else {
            gameState.value = 'nthRound';
        }
    }

    //ChatGPT znovu pomohl, moje původní verze byla nefunkční a byl jsem zoufalý
    function isInvalidSelection(selected) {
        const count = {};
        selected.forEach(d => { // d - is short for dice
            count[d.value] = (count[d.value] || 0) + 1;
        });

        const specialScore = detectSpecialCombos(count);
        if (specialScore > 0) return false;

        for (let val in count) {
            const v = parseInt(val);
            const times = count[val];

            if (times >= 3 || v === 1 || v === 5) continue;
            return true; // found an invalid dice
        }

        return false; // no invalid dice
    }

    //Storing dice into bank logic//
    // note to self - filter takes everything that is true to set condition in the ()
    function bankSelectedDice() {
        const selected = result.value.filter(diceSelection => diceSelection.isSelected);

        if (selected.length === 0) {
            warningState.value = 'on';
            warningMsg.value = 'You must select at least one dice';
            return;
        }

        if (isInvalidSelection(selected)) {
            warningState.value = 'on';
            warningMsg.value = 'You can\'t bank dice that have no value';
            score.value = 0;
            return;
        }

        scoreLogic(); // calculate the actual score now that it's valid

        selected.forEach(diceSelection => {
            storedDice.value.push(diceSelection);
        });

        // add selected dice score to the bank
        scoreBanked.value += score.value;
        score.value = 0;
        warningState.value = 'off';
        shuffleDice(true);
    }

    //ChatGPT tohle napsal celé
    //Původní verze normálního skóre jsem dokázal udělat částečně funkční,
    //ale pak nastaly speciální komba a jiné potíže a to jsem nedokázal vyřešit.
    function scoreLogic() {
    const selected = result.value.filter(diceSelection => diceSelection.isSelected);
    const count = {};

    selected.forEach(diceSelection => {
        count[diceSelection.value] = (count[diceSelection.value] || 0) + 1;
    });

    // Check special combinations first
    const specialResult = detectSpecialCombos(count);

    if (specialResult.score > 0) {
        // Check for invalid extra dice (not part of combo and not 1 or 5)
        const comboCount = {};
        specialResult.matched.forEach(val => {
            comboCount[val] = (comboCount[val] || 0) + 1;
        });

        const extraDice = selected.filter(die => {
            const val = die.value;
            const isOneOrFive = val === 1 || val === 5;
            const selectedTimes = count[val];
            const comboTimes = comboCount[val] || 0;

            // allow scoring 1s and 5s even if not in combo
            if (isOneOrFive) return false;

            // if selected more than used in combo, it's invalid
            return selectedTimes > comboTimes;
        });

        if (extraDice.length > 0) {
            score.value = 0;
            return;
        }

        // Add bonus for extra 1s and 5s (not already in combo)
        let bonus = 0;
        const bonus1s = (count[1] || 0) - (comboCount[1] || 0);
        const bonus5s = (count[5] || 0) - (comboCount[5] || 0);
        if (bonus1s > 0) bonus += bonus1s * 100;
        if (bonus5s > 0) bonus += bonus5s * 50;

        score.value = specialResult.score + bonus;
        return;
    }

    // --- Regular scoring ---
    let total = 0;

    for (let value in count) {
        const times = count[value];
        const val = parseInt(value);

        if (times >= 3) {
            let base = (val === 1) ? 1000 : val * 100;
            let multiplier = 2 ** (times - 3);
            total += base * multiplier;
        } else {
            if (val === 1) total += times * 100;
            else if (val === 5) total += times * 50;
        }
    }

    // ❌ If score is 0 and a non-scoring die is selected, invalidate it
    const includesInvalidDie = selected.some(die => {
        const val = die.value;
        const times = count[val];
        return (
            val !== 1 &&
            val !== 5 &&
            times < 3
        );
    });

    if (includesInvalidDie) {
        score.value = 0;
        return;
    }

    score.value = total;
}

    // tahle funkce je celá ChatGPT - nevěděl jsem si s tím rady i když jsem se snažil
    function detectSpecialCombos(countObj) {
        const keys = Object.keys(countObj).map(Number).sort((a, b) => a - b);

        // 2–6 straight
        const isStraight2to6 = keys.length === 5 && keys[0] === 2 && keys[4] === 6;
        if (isStraight2to6) return { score: 750, matched: [2, 3, 4, 5, 6] };

        const isStraight1to5 = keys.length === 5 && keys.every((v, i) => v === i + 1);
        if (isStraight1to5) return { score: 500, matched: [1, 2, 3, 4, 5] };

        const isStraight1to6 = keys.length === 6 && keys.every((v, i) => v === i + 1);
        if (isStraight1to6) return { score: 1500, matched: [1, 2, 3, 4, 5, 6] };

        // 3 pairs
        const values = Object.values(countObj);
        const pairCount = values.filter(v => v === 2).length;
        if (pairCount === 3) {
            let matched = [];
            for (let val in countObj) {
                if (countObj[val] === 2) {
                    matched.push(Number(val), Number(val));
                }
            }
            return { score: 750, matched };
        }

        return { score: 0, matched: [] };
    }

    // Dice Toggle and score calculation
    function diceSelectToggle(die) {
        die.isSelected = !die.isSelected
    }

    function endRound() {
        const selected = result.value.filter(diceSelection => diceSelection.isSelected);

        if (gameState.value !== 'FARKLE' && selected.length === 0) {
            warningState.value = 'on';
            warningMsg.value = 'Pick at least one dice to end your round';
            return;
        }

        if (gameState.value !== 'FARKLE' && isInvalidSelection(selected)) {
            warningState.value = 'on';
            warningMsg.value = 'You can\'t end the round with dice that have no value';
            score.value = 0;
            return;
        }

        if (currentPlayer.value === 'player1') {
            pOneScore.value += scoreBanked.value + score.value;
            currentPlayer.value = 'player2';
        } else {
            pTwoScore.value += scoreBanked.value + score.value;
            currentPlayer.value = 'player1';
        }

        scoreBanked.value = 0;
        score.value = 0;
        storedDice.value = [];
        result.value = [];
        warningState.value = 'off';
        
        checkWinCondition();
        if (gameState.value !== 'gameOver') {
            shuffleDice(true);
        }
    }

    function checkWinCondition() {
        if (pOneScore.value >= scoreToPlay.value) {
                warningState.value = 'on';
                warningMsg.value = 'player 1 has won';
                gameState.value = 'gameOver';
        }
        else if (pTwoScore.value >= scoreToPlay.value) {
            warningState.value = 'on';
            warningMsg.value = 'player 2 has won';
            gameState.value = 'gameOver';
        }
    }

    function gameRestart() {
        gameState.value = 'start';
        gameTitle.value = 'FARKLE'
        pOneScore.value = 0;
        pTwoScore.value = 0;
        warningState.value = 'off';
        if (warningMsg.value === 'player 1 has won') {
            currentPlayer.value = 'player1';
        }
        else {
            currentPlayer.value = 'player2';
        }
    }
</script>


<template>
    <div class="entire-game">
        <div class="player" :class="currentPlayer === 'player1' ? 'player-active' : ''">
            <h3>PLAYER 1</h3>
            <h1>{{ pOneScore }}</h1>
        </div>
        <div class="farkle-container">
            <div class="score-state-container">
                <h1>{{ gameTitle }}</h1>
                <h2>{{ score }}</h2>
            </div>
            <div class="game-container">
                <div class="play-dices">
                    <div class="warning-message-container">
                        <h5 class="warning-message" v-if="warningState ==='on'">{{ warningMsg }}</h5>
                    </div>
                    <div class="dice-container">
                        <h4 class="start-game" v-if="gameState === 'start'">Press start to play</h4>
                        <div class="dice" v-for="dice in result" @click="() => {if (gameState !== 'FARKLE') {diceSelectToggle(dice);scoreLogic();}}" :class="dice.isSelected ? 'dice-selected' : 'dice'">
                            <img :src="dice.img">
                        </div>
                    </div>
                </div>
                <div class="bank-container">
                    <div class="bank-score">
                        <h3>BANKED SCORE</h3>
                        <h4>{{ scoreBanked }}</h4>
                    </div>
                    <div class="stored-dice-container">
                        <div class="stored-dice" v-for="dice in storedDice">
                            <img :src="dice.img">
                        </div>
                    </div>
                </div>
            </div>
            <div class="buttons-container">
                <button v-if="gameState ==='start'" @click="shuffleDice(false)">START GAME</button>
                <button v-if="['firstRound', 'nthRound'].includes(gameState)" @click="bankSelectedDice">BANK SELECTED DICE</button>
                <button v-if="['firstRound', 'nthRound', 'FARKLE'].includes(gameState)" @click="endRound">END ROUND</button>
                <button v-if="gameState ==='gameOver'" @click="gameRestart">RESTART GAME</button>
            </div>
        </div>
        <div class="player" :class="currentPlayer === 'player2' ? 'player-active' : ''">
            <h3>PLAYER 2</h3>
            <h1>{{ pTwoScore }}</h1>
        </div>
    </div>
</template>
    

<style scoped>
    /* Font sytling */
    h1 {
        font-size: 4em;
        font-weight: 600;
        font-style: italic;
        line-height: 1;
        margin-top: 0;
        margin-bottom: 24px;
    }
    h2 {
        color: #0277FA;
        font-size: 4em;
        font-weight: 400;
        font-style: normal;
        line-height: 1;
        margin-top: 0;
        margin-bottom: 24px;
    }
    h3 {
        font-size: 2em;
        font-weight: 600;
        font-style: italic;
        line-height: 1;
        margin-top: 0;
        margin-bottom: 8px;
    }
    h4 {
        color: #0277FA;
        font-size: 2.5em;
        font-weight: 400;
        font-style: normal;
        line-height: 1;
        margin-top: 0;
        margin-bottom: 16px;
    }
    h5 {
        color: #101010;
        font-size: 1.5em;
        font-weight: 400;
        font-style: normal;
        line-height: 1;
        margin-top: 0;
        margin-bottom: 16px;
        color: #E22D3E;
    }
    h6 {
        color: #101010;
        font-size: 1.2em;
        font-weight: 400;
        font-style: normal;
        line-height: 1;
        margin-top: 0;
        margin-bottom: 16px;
    }

    /*Buttons*/
    .buttons-container {
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 15px;
    }
    button {
        border-radius: 8px;
        padding: 0.6em 1.2em;
        font-size: 1em;
        font-weight: 500;
        font-family: Helvetica;
        background-color: #101010;
        cursor: pointer;
        transition: 0.2s;
        }
    button:hover {
        background-color: #0277FA;
    }

    /*Board Styling*/
    .entire-game {
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: center;
        width: 100vw;
    }
    .player {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        margin: 0 auto;
        transform: translateY(0px);
        transition: transform 0.4s cubic-bezier(.41,.01,0,1.01), color 0.4s ease;
    }

    .player-active {
        color: #0277FA !important;
        transform: translateY(-20px);
    }

    /* Game Styling */
    .start-game {
        color: #0277FA;
        font-size: 2em;
        font-weight: 400;
        font-style: italic;
        line-height: 1;
        margin-top: 0;
        margin-bottom: 16px;
    }
    .score-state-container {
        display: flex;
        flex-direction: column;
        margin-bottom: 32px;
    }
    .game-container {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 70px;
        margin-bottom: 80px;
    }
    .play-dices {
        display: flex;
        flex-direction: column;
        gap: 10px;
        align-items: center;
        justify-content: center;
    }
    .dice-container {
        height: 70px;
        display: flex;
        align-items: center;
        gap: 15px;
    }
    .dice {
        transform: translateY(0px);
        height: 70px;
        outline: 0px solid #0277FA;
        border-radius: 9px;
        transition: .12s cubic-bezier(.47,.07,.32,1);
    }
    .dice img {
        width: 70px;
    }
    .dice:hover {
        transform: translateY(-7px);
    }
    .dice-selected {
        outline: 5px solid #0277FA;
        border-radius: 9px;
        transition: .12s cubic-bezier(.47,.07,.32,1);
    }
    .bank-container {
        display: flex;
        flex-direction: column;
        align-items: center;
    }
    .bank-score {
        margin-bottom: 16px;
    }
    .stored-dice-container {
        height: 40px;
        display: flex;
        gap: 10px;
    }
    .stored-dice img {
        width: 40px;
    }

    /*Warning Message*/
    .warning-message-container {
        height: 30px;
        display: flex;
        align-items: center;
        justify-content: center;
    }
    .warning-message {
        animation: up .45s cubic-bezier(.09,.71,.32,1);
    }
    @keyframes up {
        0% {
            transform: translateY(20px);
            opacity: 0%;
        }
        60% {
            opacity: 100%;
        }
        100% {
            transform: translateY(0px);
        }
    }
</style>