<script lang="ts">
	import { each } from 'svelte/internal';

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
	}
	interface Letter {
		text: string;
		state: LetterState;
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
		// Tracks the current try index.
		currentTryIndex: number;
		// Tracks the current letter index.
		currentLetterIndex: number;
	}

	function createWordle(): Wordle {
		const tries: Try[] = [];
		for (let i = 0; i < NUM_TRIES; i++) {
			const letters: Letter[] = [];
			for (let j = 0; j < WORD_LENGTH; j++) {
				letters.push({ text: '', state: LetterState.PENDING });
			}
			tries.push({ letters });
		}

		return {
			tries,
			currentTryIndex: 0,
			currentLetterIndex: 0
		};
	}
	const wordle = createWordle();

	function handleKeydownEvent(e: KeyboardEvent) {
		handleClickKey(e.key);
	}
	function handleClickKey(key: string) {
		// If key is a letter, update text in the corresponding letter object.
		if (LETTERS[key.toLowerCase()]) {
			setLetter(key.toLowerCase());
			wordle.currentLetterIndex++;
			if (wordle.currentLetterIndex >= WORD_LENGTH) {
				// TODO: Need to reimplement this. Automaticaly move to the next try for now.
				wordle.currentLetterIndex = 0;
				wordle.currentTryIndex++;
			}
		}
	}

	function setLetter(letter: string) {
		wordle.tries[wordle.currentTryIndex].letters[wordle.currentLetterIndex].text = letter;
	}
</script>

<svelte:window on:keydown={handleKeydownEvent} />
<div class="w-full h-full flex flex-col items-center">
	<div
		class="flex items-center justify-center font-bold text-4xl w-full h-[54px] border-b border-b-gray-300"
	>
		My Wordle
	</div>
	<div class="flex-1 mt-2">
		<div>
			{#each wordle.tries as tr}
				<div class="flex gap-1 mb-1">
					{#each tr.letters as letter}
						<div
							class="flex items-center justify-center w-[64px] h-[64px] text-[32px] font-bold uppercase border-2 box-border border-gray-300"
						>
							{letter.text}
						</div>
					{/each}
				</div>
			{/each}
		</div>
	</div>
	<div class="mb-2">Keyboard</div>
</div>
