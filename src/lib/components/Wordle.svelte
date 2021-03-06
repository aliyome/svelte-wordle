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
		bouncing: boolean;
	}
	const enum LetterState {
		// before the current try is submitted.
		PENDING,
		// you know.
		WRONG,
		// letter in word but position is wrong.
		PARTIAL_MATCH,
		// letter and position are all correct.
		FULL_MATCH
	}

	// Keyboard rows
	const KEYBOARD_ROWS = [
		['Q', 'W', 'E', 'R', 'T', 'Y', 'U', 'I', 'O', 'P'],
		['A', 'S', 'D', 'F', 'G', 'H', 'J', 'K', 'L'],
		['Enter', 'Z', 'X', 'C', 'V', 'B', 'N', 'M', 'Backspace']
	];

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
		// Stores the state for the keyboard key indexed by keys.
		readonly currentLetterStates: Record<string, LetterState>;
		// Tracks the number of submitted tries.
		numSubmittedTries: number;
		// Tracks the current try index.
		currentTryIndex: number;
		// Tracks the current letter index.
		currentLetterIndex: number;
		// Won or not.
		won: boolean;
	}

	function createWordle(): Wordle {
		// Populate initial state of "tries"
		const tries: Try[] = [];
		for (let i = 0; i < NUM_TRIES; i++) {
			const letters: Letter[] = [];
			for (let j = 0; j < WORD_LENGTH; j++) {
				letters.push({ text: '', state: LetterState.PENDING, flipping: false, bouncing: false });
			}
			tries.push({ letters, shaking: false });
		}

		// Get a target word from the word list.
		const targetWord = WORDS[Math.floor(Math.random() * WORDS.length)].toLowerCase();

		// Generate letter counts for target word.
		const targetWordLetterCounts: Record<string, number> = {};
		for (const letter of targetWord) {
			if (!targetWordLetterCounts[letter]) {
				targetWordLetterCounts[letter] = 1;
			} else {
				targetWordLetterCounts[letter]++;
			}
		}

		// Generate initial letter states.
		const currentLetterStates: Record<string, LetterState> = {};
		for (const letter of KEYBOARD_ROWS.flat().map((x) => x.toLowerCase())) {
			currentLetterStates[letter] = LetterState.PENDING;
		}

		return {
			tries,
			targetWord,
			targetWordLetterCounts,
			currentLetterStates,
			numSubmittedTries: 0,
			currentTryIndex: 0,
			currentLetterIndex: 0,
			won: false
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

	// Controls the display of the share dialog
	let isShowShareDialog = false;

	function handleKeydownEvent(e: KeyboardEvent) {
		handleClickKey(e.key);
	}
	function handleClickKey(key: string) {
		// Don't process key down when user has won the game or running out of tries.
		if (wordle.won || wordle.numSubmittedTries >= NUM_TRIES) {
			return;
		}

		// If key is a letter, update text in the corresponding letter object.
		if (LETTERS[key.toLowerCase()]) {
			// Only allow typing letters in the current try.
			// Don't go over if the current try has not been submitted.
			if (wordle.currentLetterIndex >= WORD_LENGTH) {
				if (wordle.currentTryIndex < wordle.numSubmittedTries) {
					wordle.currentLetterIndex = 0;
					wordle.currentTryIndex++;
					setLetter(key.toLowerCase());
					wordle.currentLetterIndex++;
				}
			} else {
				setLetter(key.toLowerCase());
				wordle.currentLetterIndex++;
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

		// Check if the current try matches the target word.

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

		// Store to keyboard keys states.
		// Do this after the current try has been submitted
		// and the animation above is done.
		for (const letter of tr.letters) {
			const actual = letter.text.toLowerCase();
			const currentStoredState = wordle.currentLetterStates[actual];
			const targetStoredState = letter.state;
			// This allows override state with better result.
			// For example, if "A" was partial match in previous try,
			// and becomes full match in the current try,
			// we update the key state to the full match
			// (because its enum value is larger).
			if (targetStoredState > currentStoredState) {
				wordle.currentLetterStates[actual] = targetStoredState;
			}
		}

		// Move to next try
		wordle.numSubmittedTries++;

		// Check if all letters in the current try are correct.
		if (states.every((state) => state === LetterState.FULL_MATCH)) {
			showInfoMessage('You did it!');
			wordle.won = true;
			// Bounce animation
			for (let i = 0; i < tr.letters.length; i++) {
				wordle.tries[wordle.currentTryIndex].letters[i].bouncing = true;
				await sleep(160);
			}

			showShare();
			return;
		}

		// Running out of tries. Show correct answer.
		if (wordle.numSubmittedTries >= NUM_TRIES) {
			showInfoMessage(wordle.targetWord.toUpperCase(), false);
			showShare();
		}
	}

	async function shakeCurrentRow() {
		wordle.tries[wordle.currentTryIndex].shaking = true;
		await sleep(500);
		wordle.tries[wordle.currentTryIndex].shaking = false;
	}

	async function showInfoMessage(msg: string, hide = true) {
		isShownMessageSnackbar = true;
		infoMessage = msg;
		if (!hide) {
			return;
		}

		// Hide after 2s.
		await sleep(2000);
		// When isShownMessageSnackbar is false, the fadeout transition will be played.
		// Remain info message between fade out transition.
		isShownMessageSnackbar = false;
		await sleep(500);
		infoMessage = '';
	}

	async function showShare() {
		await sleep(1500);
		isShowShareDialog = true;
	}

	function handleClickShare() {
		// 🟩 🟨 ⬛ ⬜
		// Copy results into clipboard
		let clipboardContent = '';
		for (const tr of wordle.tries) {
			for (const letter of tr.letters) {
				switch (letter.state) {
					case LetterState.FULL_MATCH:
						clipboardContent += '🟩';
						break;
					case LetterState.PARTIAL_MATCH:
						clipboardContent += '🟨';
						break;
					case LetterState.WRONG:
						clipboardContent += '⬛';
						break;
					default:
						break;
				}
			}
			clipboardContent += '\n';
		}
		navigator.clipboard.writeText(clipboardContent);
		isShowShareDialog = false;
		showInfoMessage('Copied result to clipboard');
	}
</script>

<svelte:window on:keydown={handleKeydownEvent} />
<div class="relative w-full h-full flex flex-col items-center">
	<div
		class="flex items-center justify-center font-bold text-4xl w-full h-[54px] border-b border-b-gray-300"
	>
		My Wordle
	</div>
	<div class="flex-1 mt-2 flex flex-col justify-center">
		<div>
			{#each wordle.tries as tr}
				<div class="flex gap-1 mb-1" class:shake-anim={tr.shaking}>
					{#each tr.letters as letter}
						<div
							class="flex items-center justify-center w-[64px] h-[64px] text-[32px] font-bold uppercase border-2 box-border border-gray-300 transition-transform duration-[180ms]"
							class:border-gray-600={letter.text !== ''}
							class:letter-pop-anim={letter.text !== ''}
							class:letter-flip-anim={letter.flipping}
							class:bouncing-anim={letter.bouncing}
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
	<div class="mb-2 flex flex-col items-center">
		{#each KEYBOARD_ROWS as row}
			<div class="flex items-center mt-1 gap-1">
				{#each row as key}
					<button
						on:click={() => handleClickKey(key)}
						type="button"
						class="flex items-center justify-center min-w-[30px] h-[50px] bg-gray-300 rounded font-bold"
						class:px-1={['Enter', 'Backspace'].includes(key)}
						class:text-xs={['Enter', 'Backspace'].includes(key)}
						class:!bg-green-600={wordle.currentLetterStates[key.toLowerCase()] ===
							LetterState.FULL_MATCH}
						class:!bg-orange-600={wordle.currentLetterStates[key.toLowerCase()] ===
							LetterState.PARTIAL_MATCH}
						class:!bg-gray-600={wordle.currentLetterStates[key.toLowerCase()] === LetterState.WRONG}
						class:!text-gray-100={wordle.currentLetterStates[key.toLowerCase()] !==
							LetterState.PENDING}>{key}</button
					>
				{/each}
			</div>
		{/each}
	</div>

	<!-- snackbar for info message -->
	<div
		class="absolute top-24 px-6 py-4 bg-black text-white rounded-lg font-bold transition-opacity duration-500 opacity-0"
		class:!opacity-1={isShownMessageSnackbar}
	>
		{infoMessage}
	</div>

	<!-- dialog for share result -->
	<div
		class="absolute w-full h-full bg-white/50 flex items-center justify-center transition-opacity duration-100 opacity-0 invisible"
		class:opacity-1={isShowShareDialog}
		class:visible={isShowShareDialog}
	>
		<div
			class="w-[300px] h-[300px] bg-white flex items-center justify-center rounded border shadow-xl transition-all duration-300"
			class:opacity-0={!isShowShareDialog}
			class:opacity-1={isShowShareDialog}
			class:translate-y-10={!isShowShareDialog}
			class:translate-y-0={isShowShareDialog}
		>
			<button
				class="bg-green-500 text-white font-bold px-4 py-2 rounded"
				on:click={handleClickShare}
			>
				Share
			</button>
		</div>
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

	/* Again, got this from wordle css:) */
	@keyframes bounce {
		0%,
		20% {
			transform: translateY(0);
		}
		40% {
			transform: translateY(-30px);
		}
		50% {
			transform: translateY(5px);
		}
		60% {
			transform: translateY(-15px);
		}
		80% {
			transform: translateY(2px);
		}
		100% {
			transform: translateY(0);
		}
	}

	.bouncing-anim {
		animation: bounce 1s ease-in-out;
	}
</style>
