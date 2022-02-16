<script lang="ts">
	import { words as WORDS } from './words';

	// Util
	const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

	// Maximum length of the word
	const WORD_LENGTH = 5;
	// Maximum numbers of tries
	const NUM_TRIES = 6;

	// Letter map[a-z]
	const LETTERS = (() => {
		const ret: Record<string, boolean> = {};
		for (let charCode = 97; charCode < 97 + 26; charCode++) {
			ret[String.fromCharCode(charCode)] = true;
		}
		return ret;
	})();

	interface Try {
		letters: Letter[];
		shaking: boolean;
	}
	interface Letter {
		text: string;
		state: LetterState;
		flipping: boolean;
	}
	const enum LetterState {
		// you know.
		WRONG,
		// letter in word but position is wrong.
		PARTIAL_MATCH,
		// letter and position are all correct.
		FULL_MATCH,
		// before the current try is submitted.
		PENDING
	}

	interface Wordle {
		// Stores all tries.
		// One try is one row in the UI.
		readonly tries: Try[];
		// Store the target word
		readonly targetWord: string;
		// Store the count for each letter from the target word.
		//
		// For example, if the target word is "happy", then this record will look like:
		// { h:1, a:1, p:2, y:1 }
		readonly targetWordLetterCounts: Record<string, number>;
		// Tracks the number of submitted tries.
		numSubmittedTries: number;
		// Tracks the current try index.
		currentTryIndex: number;
		// Tracks the current letter index.
		currentLetterIndex: number;
	}

	function createWordle(): Wordle {
		// Populate initial state of "tries"
		const tries: Try[] = [];
		for (let i = 0; i < NUM_TRIES; i++) {
			const letters: Letter[] = [];
			for (let j = 0; j < WORD_LENGTH; j++) {
				letters.push({ text: '', state: LetterState.PENDING, flipping: false });
			}
			tries.push({ letters, shaking: false });
		}

		// Get a target word from the word list.
		// const targetWord = WORDS[Math.floor(Math.random() * WORDS.length)].toLowerCase();
		// TODO: Remove this.
		const targetWord = 'happy';

		// TODO: Remove this cheat log
		console.log('target word: ', targetWord);

		// Generate letter counts for target word.
		const targetWordLetterCounts: Record<string, number> = {};
		for (const letter of targetWord) {
			if (!targetWordLetterCounts[letter]) {
				targetWordLetterCounts[letter] = 1;
			} else {
				targetWordLetterCounts[letter]++;
			}
		}

		return {
			tries,
			targetWord,
			targetWordLetterCounts,
			numSubmittedTries: 0,
			currentTryIndex: 0,
			currentLetterIndex: 0
		};
	}
	const wordle = createWordle();

	// Message shown in the message snackbar.
	let infoMessage = '';
	// Controls the display of the snackbar.
	// The reason why we have separate infoMessage and isShownMessageSnackbar is
	// because we want to clear the infoMessage after a while after the isShownMessageSnackbar is set to false.
	// When the isShownMessageSnackbar becomes false, the fade-out transition will be played,
	// and we want to keep the infoMessage intact while it plays.
	let isShownMessageSnackbar = false;

	function handleKeydownEvent(e: KeyboardEvent) {
		handleClickKey(e.key);
	}
	function handleClickKey(key: string) {
		// If key is a letter, update text in the corresponding letter object.
		if (LETTERS[key.toLowerCase()]) {
			setLetter(key.toLowerCase());
			wordle.currentLetterIndex++;
			// Only allow typing letters in the current try.
			// Don't go over if the current try has not been submitted.
			if (wordle.currentLetterIndex >= WORD_LENGTH) {
				if (wordle.currentTryIndex < wordle.numSubmittedTries) {
					wordle.currentLetterIndex = 0;
					wordle.currentTryIndex++;
				}
			}
		}
		// Handle backspace
		else if (key === 'Backspace') {
			wordle.currentLetterIndex = Math.max(--wordle.currentLetterIndex, 0);
			setLetter('');
		}
		// Submit the current try and check.
		else if (key === 'Enter') {
			checkCurrentTry();
		}
	}

	function setLetter(letter: string) {
		wordle.tries[wordle.currentTryIndex].letters[wordle.currentLetterIndex].text = letter;
	}

	async function checkCurrentTry() {
		// Check if user has typed all letters.
		const tr = wordle.tries[wordle.currentTryIndex];
		if (tr.letters.some((letter) => letter.text === '')) {
			showInfoMessage('Not enough letters');
			await shakeCurrentRow();
			return;
		}

		// Check if the current try is a word in the list.
		const wordOnCurrentTry = tr.letters.map((letter) => letter.text).join('');
		if (!WORDS.includes(wordOnCurrentTry)) {
			showInfoMessage('Not in word list');
			await shakeCurrentRow();
			return;
		}

		// // Check if the current try matches the target word.
		// if (wordOnCurrentTry !== wordle.targetWord) {
		// }

		// Store the check results
		const states: LetterState[] = [];
		// Clone the count map. Need to use it in every check with the initial values.
		const targetWordLetterCounts = { ...wordle.targetWordLetterCounts };
		for (let i = 0; i < WORD_LENGTH; i++) {
			const expected = wordle.targetWord[i];
			const currentLetter = tr.letters[i];
			const actual = currentLetter.text.toLowerCase();
			let state = LetterState.WRONG;
			// Need to make sure only performs the check when the letter has not been checked before.
			//
			// For example, if the target word is "happy",
			// then the first "a" user types should be checked, but the second "a" should not,
			// because there is no more "a" left in the target word that has not not been checked.
			if (expected === actual && targetWordLetterCounts[actual] > 0) {
				targetWordLetterCounts[actual]--;
				state = LetterState.FULL_MATCH;
			} else if (wordle.targetWord.includes(actual) && targetWordLetterCounts[actual] > 0) {
				targetWordLetterCounts[actual]--;
				state = LetterState.PARTIAL_MATCH;
			}
			states.push(state);
		}

		// Animate
		for (let i = 0; i < tr.letters.length; i++) {
			// "Flip" the letter, apply the result (and update the style), then unflip it.
			wordle.tries[wordle.currentTryIndex].letters[i].flipping = true;
			// Wait for the flip animation to finish.
			await sleep(180);
			// Update state. This will also update styles.
			wordle.tries[wordle.currentTryIndex].letters[i].state = states[i];
			// Unfold
			wordle.tries[wordle.currentTryIndex].letters[i].flipping = false;
			await sleep(180);
		}
	}

	async function shakeCurrentRow() {
		wordle.tries[wordle.currentTryIndex].shaking = true;
		await sleep(500);
		wordle.tries[wordle.currentTryIndex].shaking = false;
	}

	async function showInfoMessage(msg: string) {
		isShownMessageSnackbar = true;
		infoMessage = msg;
		// Hide after 2s.
		await sleep(2000);
		// When isShownMessageSnackbar is false, the fadeout transition will be played.
		// Remain info message between fade out transition.
		isShownMessageSnackbar = false;
		await sleep(500);
		infoMessage = '';
	}
</script>

<svelte:window on:keydown={handleKeydownEvent} />
<div class="relative w-full h-full flex flex-col items-center">
	<div
		class="flex items-center justify-center font-bold text-4xl w-full h-[54px] border-b border-b-gray-300"
	>
		My Wordle
	</div>
	<div class="flex-1 mt-2">
		<div>
			{#each wordle.tries as tr}
				<div class="flex gap-1 mb-1" class:shake-anim={tr.shaking}>
					{#each tr.letters as letter}
						<div
							class="flex items-center justify-center w-[64px] h-[64px] text-[32px] font-bold uppercase border-2 box-border border-gray-300 transition-transform duration-[180ms]"
							class:border-gray-600={letter.text !== ''}
							class:letter-pop-anim={letter.text !== ''}
							class:letter-flip-anim={letter.flipping}
							class:bg-green-600={letter.state === LetterState.FULL_MATCH}
							class:bg-orange-600={letter.state === LetterState.PARTIAL_MATCH}
							class:bg-gray-600={letter.state === LetterState.WRONG}
							class:text-gray-100={letter.state !== LetterState.PENDING}
							class:scale-y-0={letter.flipping}
						>
							{letter.text}
						</div>
					{/each}
				</div>
			{/each}
		</div>
	</div>
	<div class="mb-2">Keyboard</div>

	<!-- snackbar for info message -->
	<div
		class="absolute top-24 px-6 py-4 bg-black text-white rounded-lg font-bold transition-opacity duration-500"
		class:opacity-0={!isShownMessageSnackbar}
		class:opacity-1={isShownMessageSnackbar}
	>
		{infoMessage}
	</div>
</div>

<style>
	@keyframes letter-pop {
		0% {
			transform: scale(1);
		}
		50% {
			transform: scale(1.2);
		}
		100% {
			transform: scale(1);
		}
	}
	.letter-pop-anim {
		animation: letter-pop 120ms ease-in-out;
	}

	/* Stole from wordle css. */
	@keyframes shake {
		10%,
		90% {
			transform: translateX(-1px);
		}

		20%,
		80% {
			transform: translateX(2px);
		}

		30%,
		50%,
		70% {
			transform: translateX(-4px);
		}

		40%,
		60% {
			transform: translateX(4px);
		}
	}
	.shake-anim {
		animation: shake 500ms ease-in-out;
	}

	.flip-anim {
	}
</style>
