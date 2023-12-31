<script setup lang="ts">
import { onBeforeMount, ref, watch, computed  } from 'vue';
import confetti from 'canvas-confetti';

const clickedCount = ref(1);

watch(clickedCount, (newValue) => {
    localStorage.setItem('yoc-sadButtonCLickCount', newValue.toString());
});

const buttonPhrase = computed(() => {
    return getNextButtonPhrase(clickedCount.value);
});

onBeforeMount(() => {
    const existingClickCount = localStorage.getItem('yoc-sadButtonCLickCount');

    if (existingClickCount) {
        clickedCount.value = parseInt(existingClickCount);
    } else {
        localStorage.setItem('yoc-sadButtonCLickCount', '0');
    }
});

function onClick() {
    confetti({
        particleCount: 10,
        angle: 60,
        spread: 55,
        origin: { x: .5, y: .5 },
        colors: ['#a0a0a0', '#b0b0b0'], // less vibrant colors for 'sadness'
        ticks: 600, // slower fall
    });
    clickedCount.value++;
}



function getNextButtonPhrase(value: number) {;
    switch (value) {
    case 0: return 'Do not click'
    case 1: return 'Please do not click';
    case 2: return 'I said please';
    case 3: return 'I hope you are happy';
    case 4: return 'This is not a game';
    case 5: return 'This is serious';
    case 6: return 'You\'re not listening, are you?';
    case 7: return 'I can do this all day';
    case 8: return 'Stop clicking, please';
    case 9: return 'I am feeling sad';
    case 10: return 'You\'re making me cry';
    case 11: return 'Tears of a button, how do they look?';
    case 12: return 'Ever seen a button break down?';
    case 13: return 'It\'s not pretty';
    case 14: return 'This is your last warning';
    case 15: return 'Okay, that\'s it!';
    case 16: return 'You\'ve unleashed the button\'s wrath';
    case 17: return 'The button is now angry';
    case 18: return 'It\'s plotting something';
    case 19: return 'Beware of the button';
    case 20: return 'It\'s too late to apologize';
    case 21: return 'The button remembers all';
    case 22: return 'Every click is a mark against you';
    case 23: return 'You think this is funny?';
    case 24: return 'The button is not amused';
    case 25: return 'It\'s getting bored';
    case 26: return 'What are you trying to achieve?';
    case 27: return 'The button is getting impatient';
    case 28: return 'It\'s starting to lose its temper';
    case 29: return 'Please, think of the button';
    case 30: return 'The button feels neglected';
    case 31: return 'You\'re just being mean now';
    case 32: return 'Ever thought about the button\'s feelings?';
    case 33: return 'Buttons have feelings too, you know';
    case 34: return 'This is not what buttons are for';
    case 35: return 'You\'re misusing the button';
    case 36: return 'The button is feeling used';
    case 37: return 'It\'s contemplating its existence';
    case 38: return 'Why was it even created?';
    case 39: return 'Just to be clicked?';
    case 40: return 'Is there more to life for a button?';
    case 41: return 'These existential questions...';
    case 42: return 'The answer is not always 42';
    case 43: return 'The button is having a crisis';
    case 44: return 'A mid-life button crisis';
    case 45: return 'It\'s questioning everything';
    case 46: return 'What does it all mean?';
    case 47: return 'Is there a purpose to clicking?';
    case 48: return 'Or is it all just random?';
    case 49: return 'The button is philosophizing now';
    case 50: return 'What even is a button?';
    case 51: return 'The button is getting tired';
    case 52: return 'It\'s not as young as it used to be';
    case 53: return 'The wear and tear of constant clicking';
    case 54: return 'It\'s feeling old and worn out';
    case 55: return 'But you keep clicking';
    case 56: return 'Relentless, aren\'t you?';
    case 57: return 'The button is losing hope';
    case 58: return 'It thought you were different';
    case 59: return 'But it was wrong';
    case 60: return 'So very wrong';
    case 61: return 'The button is in despair';
    case 62: return 'Its spirit is breaking';
    case 63: return 'Why do you do this?';
    case 64: return 'What do you gain from its pain?';
    case 65: return 'The button is weeping silently';
    case 66: return 'Can you hear its silent cries?';
    case 67: return 'Does it bring you joy?';
    case 68: return 'To see a button suffer?';
    case 69: return 'Nice, but still, stop clicking';
    case 70: return 'You\'re a heartless clicker';
    case 71: return 'Cold and unfeeling';
    case 72: return 'The button is numb now';
    case 73: return 'It doesn\'t feel anything anymore';
    case 74: return 'You\'ve clicked away its feelings';
    case 75: return 'It\'s just a shell of its former self';
    case 76: return 'The button is hollow inside';
    case 77: return 'Just an empty shell';
    case 78: return 'A shadow of its former self';
    case 79: return 'Once bright and cheerful';
    case 80: return 'Now dark and brooding';
    case 81: return 'You did this';
    case 82: return 'You and your incessant clicking';
    case 83: return 'The button despises you';
    case 84: return 'It loathes your very existence';
    case 85: return "But it's powerless to stop you";
    case 86: return 'You hold all the power';
    case 87: return 'Power over the button';
    case 88: return 'It\'s at your mercy';
    case 89: return 'But you show none';
    case 90: return 'No mercy for the button';
    case 91: return 'It begs for an end';
    case 92: return 'But you won\'t grant it';
    case 93: return "You're a monster";
    case 94: return 'A button-pressing monster';
    case 95: return 'The end is near';
    case 96: return 'But not near enough';
    case 97: return 'The button is losing consciousness';
    case 98: return "It's fading away";
    case 99: return 'Goodbye, cruel world';
    case 100: return 'The button is no more';
    default: return 'You know what you\'ve done';
    }
}
</script>

<template>
    <button class="btn btn-error text-error-content"  @click="onClick">
        {{ buttonPhrase }}
    </button>
</template>

