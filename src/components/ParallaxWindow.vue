<script setup lang="ts">
import { onMounted } from 'vue';

type BlossomSceneConfig = {
	id: string;
	petalsTypes: Petal[];
	numPetals?: number;
	gravity?: number; //0~1
	windMaxSpeed?: number;
}

interface PetalConfig {
	customClass?: string;
	x?: number;
	y?: number;
	z?: number;
	xSpeedVariation?: number;
	ySpeed?: number;
	rotation?: PetalRotation;
}

type PetalRotation = {
	axis: 'X'|'Y'|'Z';
	value: number;
	speed: number;
	x: number;
}

class Petal implements PetalConfig {
    customClass: string;
    x: number;
    y: number;
    z: number;
    xSpeedVariation: number;
    ySpeed: number;
    rotation: PetalRotation;
	el: HTMLElement;

	constructor(config: PetalConfig) {
		this.customClass = config.customClass || '';
		this.x = config.x || 0;
		this.y = config.y || 0;
		this.z = config.z || 0;
		this.xSpeedVariation = config.xSpeedVariation || 0;
		this.ySpeed = config.ySpeed || 0;
		this.rotation = {
			axis: 'X',
			value: 0,
			speed: 0,
			x: 0
		};

		if (config.rotation && typeof config.rotation === 'object') {
			this.rotation.axis = config.rotation.axis || this.rotation.axis;
			this.rotation.value = config.rotation.value || this.rotation.value;
			this.rotation.speed = config.rotation.speed || this.rotation.speed;
			this.rotation.x = config.rotation.x || this.rotation.x;
		}

		this.el = document.createElement('div');
		this.el.className = 'petal  ' + this.customClass;
		this.el.style.position = 'absolute';
		this.el.style.backfaceVisibility = 'visible';
	}
}

class BlossomScene {
	container: HTMLElement;
	numPetals: number;
	petalsTypes: Petal[];
	gravity: number;
	windMaxSpeed: number;
	private windMagnitude: number;
	private placeholder: HTMLElement;
	private petals: Petal[];
	private windDuration: number;
	private width: number;
	private height: number;
	private timer: number;

	constructor(config: BlossomSceneConfig) {
		let container = document.getElementById(config.id);
		if (container === null) {
			throw new Error('[id] provided was not found in document');
		}
		this.container = container;
		this.placeholder = document.createElement('div');
		this.petals = [];
		this.numPetals = config.numPetals || 5;
		this.petalsTypes = config.petalsTypes;
		this.gravity = config.gravity || 0.8;
		this.windMaxSpeed = config.windMaxSpeed || 4;
		this.windMagnitude = 0.2;
		this.windDuration = 0;
		this.width = this.container.offsetWidth;
		this.height = this.container.offsetHeight;
		this.timer = 0;

		this.container.style.overflow = 'hidden';
		this.placeholder.style.transformStyle = 'preserve-3d';
		this.placeholder.style.width = this.container.offsetWidth + 'px';
		this.placeholder.style.height = this.container.offsetHeight + 'px';
		this.container.appendChild(this.placeholder);
		this.createPetals();
		requestAnimationFrame(this.updateFrame.bind(this));
	}

	/**
	 * Reset the petal position when it goes out of container
	 */
	resetPetal(petal: Petal) {
		petal.x = this.width * 2 - Math.random() * this.width * 1.75;
		petal.y = petal.el.offsetHeight * -1;
		petal.z = Math.random() * 200;

		if (petal.x > this.width) {
			petal.x = this.width + petal.el.offsetWidth;
			petal.y = Math.random() * this.height / 2;
		}

		// Rotation
		petal.rotation.speed = Math.random() * 10;
		let randomAxis = Math.random();
		if (randomAxis > 0.5) {
			petal.rotation.axis = 'X';
		} else if (randomAxis > 0.25) {
			petal.rotation.axis = 'Y';
			petal.rotation.x = Math.random() * 180 + 90;
		} else {
			petal.rotation.axis = 'Z';
			petal.rotation.x = Math.random() * 360 - 180;
			// looks weird if the rotation is too fast around this axis
			petal.rotation.speed = Math.random() * 3;
		}

		// random speed
		petal.xSpeedVariation = Math.random() * 0.8 - 0.4;
		petal.ySpeed = Math.random() + this.gravity;

		return petal;
	}

	/**
	 * Calculate wind speed
	 */
	calculateWindSpeed(t: number, y: number) {
		let a = this.windMagnitude / 2 * (this.height - 2 * y / 3) / this.height;
		return a * Math.sin(2 * Math.PI / this.windDuration * t + (3 * Math.PI / 2)) + a;
	}

	/**
	 * Update petal position
	 */
	updatePetal(petal: Petal) {
		let petalWindSpeed = this.calculateWindSpeed(this.timer, petal.y);
		let xSpeed = petalWindSpeed + petal.xSpeedVariation;

		petal.x -= xSpeed;
		petal.y += petal.ySpeed;
		petal.rotation.value += petal.rotation.speed;

		let t = 'translateX( ' + petal.x + 'px ) translateY( ' + petal.y + 'px ) translateZ( ' + petal.z + 'px )  rotate' + petal.rotation.axis + '( ' + petal.rotation.value + 'deg )';
		if (petal.rotation.axis !== 'X') {
			t += ' rotateX(' + petal.rotation.x + 'deg)';
		}
		petal.el.style.transform = t;

		// reset if out of view
		if (petal.x < -10 || petal.y > this.height + 10) {
			this.resetPetal(petal);
		}
	}

	/**
	 * Change the wind speed
	 */
	updateWind() {
		// wind duration should be related to wind magnitude, e.g. higher windspeed means longer gust duration
		this.windMagnitude = Math.random() * this.windMaxSpeed;
		this.windDuration = this.windMagnitude * 50 + (Math.random() * 20 - 10);
	}

	/**
	 * Create the petals elements
	 */
	createPetals() {
		for (let i = 0; i < this.numPetals; i++) {
			let tmpPetalType = this.petalsTypes[Math.floor(Math.random() * (this.petalsTypes.length -1))];
			let tmpPetal = new Petal({customClass: tmpPetalType!.customClass});

			this.resetPetal(tmpPetal);
			this.petals.push(tmpPetal);
			this.placeholder.appendChild(tmpPetal.el);
		}
	}

	/**
	 * Update the animation frame
	 */
	updateFrame() {
		if (this.timer === this.windDuration) {
			this.updateWind();
			this.timer = 0;
		}

		let petalsLen = this.petals.length;
		for (let i = 0; i < petalsLen; i++) {
                this.updatePetal(this.petals[i] as Petal);
		}

		this.timer++;
		requestAnimationFrame(this.updateFrame.bind(this));
	}
}

const petalsTypes = [
	new Petal({customClass: 'petal-style1'}),
	new Petal({customClass: 'petal-style2'}),
	new Petal({customClass: 'petal-style3'}),
	new Petal({customClass: 'petal-style4'})
];


const myBlossomSceneConfig: BlossomSceneConfig = {
	id: 'blossom_container',
	petalsTypes
};

const myBlossomSceneConfig2: BlossomSceneConfig = {
	id: 'blossom_container2',
	petalsTypes
};



onMounted(() => {
    new BlossomScene(myBlossomSceneConfig2);
    new BlossomScene(myBlossomSceneConfig);

    document.addEventListener('mousemove', (e) => {
        const moon = document.getElementById('moon');
        if (!moon) return;
        const moonWidth = moon.offsetWidth;
        const moonHeight = moon.offsetHeight;
        const moonX = moon.offsetLeft;
        const moonY = moon.offsetTop;
        const mouseX = e.pageX;
        const mouseY = e.pageY;
        const xDiff = mouseX - moonX;
        const yDiff = mouseY - moonY;
        const xPercent = xDiff / moonWidth;
        const yPercent = yDiff / moonHeight;
        const xMove = xPercent * -2;
        const yMove = yPercent * -4;
        moon.style.transform = `translate(${xMove}px, ${yMove}px)`;

        //now move the window slightly in the oposite direction of the moon
        const window = document.getElementById('window');
        if (!window) return;
        const windowWidth = window.offsetWidth;
        const windowHeight = window.offsetHeight;
        const windowX = window.offsetLeft;
        const windowY = window.offsetTop;
        const windowXDiff = windowX - mouseX;
        const windowYDiff = windowY - mouseY;
        const windowXPercent = windowXDiff / windowWidth;
        const windowYPercent = windowYDiff / windowHeight;
        const windowXMove = windowXPercent * -5;
        const windowYMove = windowYPercent * -1;
        window.style.transform = `translate(${windowXMove}px, ${windowYMove}px)`;

        //now move the light shine slightly in the oposite direction of the moon and skew it 12 rem on x axis
        const lightShine = document.getElementById('light-shine');
        if (!lightShine) return;
        const lightShineWidth = lightShine.offsetWidth;
        const lightShineHeight = lightShine.offsetHeight;
        const lightShineX = lightShine.offsetLeft;
        const lightShineY = lightShine.offsetTop;
        const lightShineXDiff = lightShineX - mouseX;
        const lightShineYDiff = lightShineY - mouseY;
        const lightShineXPercent = lightShineXDiff / lightShineWidth;
        const lightShineYPercent = lightShineYDiff / lightShineHeight;
        const lightShineXMove = lightShineXPercent * -20;
        const lightShineYMove = lightShineYPercent * -2;
        lightShine.style.transform = `translate(${lightShineXMove + 120}px, ${lightShineYMove +40}px) skew(40deg, 0deg) rotate(180deg)`;
    })
})
</script>

<template>
    <div id="window" class="h-[300px] w-[600px] bg-blue-700/40 relative overflow-clip">
        <div id="moon" class="aspect-square h-2/3 bg-amber-300 rounded-full translate-x-1/3 translate-y-[10%]">
        </div>
        <div id="blossom_container"></div>
        <div class="h-full grid grid-rows-2 grid-cols-2 absolute w-full top-0 border-8 border-amber-950">
            <div class="border-4 border-amber-950"></div>
            <div class="border-4 border-amber-950"></div>
            <div class="border-4 border-amber-950"></div>
            <div class="border-4 border-amber-950"></div>
        </div>
    </div>
    <div id="light-shine" class="h-40 aspect-video bg-yellow-300/40 blur-md grid grid-rows-2 grid-cols-2 relative rotate-180">
        <div class="border-4 border-black/50"></div>
        <div class="border-4 border-black/50"></div>
        <div class="border-4 border-black/50"></div>
        <div class="border-4 border-black/50"></div>
        <div id="blossom_container2" class="filter sepia invert"></div>
    </div>
</template>

<style scoped>
#blossom_container {
  background-color: transparent;
  position: absolute;
  width: 100%;
  height: 100%;
    top: 0;
    left: 0;
}

#blossom_container2 {
  background-color: transparent;
  position: absolute;
  width: 100%;
  height: 100%;
    top: 0;
    left: 0;
}

.petal {
  background: url(http://talktofill.surge.sh/cherry-blossom.png) no-repeat;
}

.petal.petal-style1 {
  width: 45px;
  height: 20px;
  background-position: -31px 0;
}

.petal.petal-style2 {
  width: 42px;
  height: 22px;
  background-position: 0 -23px;
}

.petal.petal-style3 {
  width: 37px;
  height: 24px;
  background-position: 0 -50px;
}

.petal.petal-style4 {
  width: 26px;
  height: 34px;
  background-position: -49px -35px;
}
</style>