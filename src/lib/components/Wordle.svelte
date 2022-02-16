<script lang="ts">
	import { each } from 'svelte/internal';

	// Maximum length of the word
	const WORD_LENGTH = 5;
	// Maximum numbers of tries
	const NUM_TRIES = 6;

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
			tries
		};
	}
	const wordle = createWordle();
</script>

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
							{letter.text}L
						</div>
					{/each}
				</div>
			{/each}
		</div>
	</div>
	<div class="mb-2">Keyboard</div>
</div>
