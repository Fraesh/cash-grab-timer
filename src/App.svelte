<script>
  const ROBBER_OPTIONS = [1, 2, 3, 4, 5, 6];
  const TIME_OPTIONS = [15, 30, 45, 60];
  const BANK_EMPLOYEES = 2;
  const TOTAL_ROUNDS = 6;
  const LANGUAGES = ['en', 'de', 'fr'];
  const LANGUAGE_LABELS = {
    en: 'English',
    de: 'Deutsch',
    fr: 'Francais',
  };
  const COPY = {
    en: {
      appEyebrow: 'Cash Grab companion',
      title: 'Turn Timer',
      lede: 'Set the crew size, pick the tempo, and keep robber and bank turns moving.',
      language: 'Language',
      robbers: 'Robbers',
      robberTeam: 'Robbers',
      bankTeam: 'Bank',
      plusBank: 'Plus 2 bank employees.',
      timePerPlayer: 'Time per figure',
      startGame: 'Start Game',
      round: 'Round',
      turn: 'Turn',
      players: 'Figures',
      backToSetup: 'Back to setup',
      start: 'Start',
      stop: 'Stop',
      done: 'Done',
      finalBell: 'Final bell',
      gameComplete: 'Game Complete',
      allRoundsFinished: (rounds) => `All ${rounds} rounds are finished.`,
      playAgain: 'Play Again',
      changeSetup: 'Change Setup',
      summaryRobbers: (time) => `Robbers: ${time}`,
      summaryBank: (time) => `Bank: ${time}`,
      summaryRounds: (rounds) => `${rounds} rounds`,
      turnLabel: (round, team) => `Round ${round}, ${team}`,
    },
    de: {
      appEyebrow: 'Cash Grab Begleiter',
      title: 'Rundentimer',
      lede: 'Legt die Crew-Groesse fest, waehlt das Tempo und haltet Bankräuber- und Bankzuege in Bewegung.',
      language: 'Sprache',
      robbers: 'Bankräuber',
      robberTeam: 'Bankräuber',
      bankTeam: 'Bank',
      plusBank: 'Plus 2 Bankangestellte.',
      timePerPlayer: 'Zeit pro Spielfigur',
      startGame: 'Spiel starten',
      round: 'Runde',
      turn: 'Zug',
      players: 'Spielfiguren',
      backToSetup: 'Zurueck zur Einrichtung',
      start: 'Start',
      stop: 'Stopp',
      done: 'Fertig',
      finalBell: 'Schlussglocke',
      gameComplete: 'Spiel beendet',
      allRoundsFinished: (rounds) => `Alle ${rounds} Runden sind beendet.`,
      playAgain: 'Noch mal',
      changeSetup: 'Setup aendern',
      summaryRobbers: (time) => `Bankräuber: ${time}`,
      summaryBank: (time) => `Bank: ${time}`,
      summaryRounds: (rounds) => `${rounds} Runden`,
      turnLabel: (round, team) => `Runde ${round}, ${team}`,
    },
    fr: {
      appEyebrow: 'Compagnon Cash Grab',
      title: 'Chrono de tour',
      lede: "Choisissez la taille de l'equipe, le rythme, et gardez les tours des braqueurs et de la banque bien cadences.",
      language: 'Langue',
      robbers: 'Braqueurs',
      robberTeam: 'Braqueurs',
      bankTeam: 'Banque',
      plusBank: 'Plus 2 employes de banque.',
      timePerPlayer: 'Temps par figurine',
      startGame: 'Commencer',
      round: 'Manche',
      turn: 'Tour',
      players: 'Figurines',
      backToSetup: 'Retour aux reglages',
      start: 'Start',
      stop: 'Stop',
      done: 'Fini',
      finalBell: 'Derniere sonnerie',
      gameComplete: 'Partie terminee',
      allRoundsFinished: (rounds) => `Les ${rounds} manches sont terminees.`,
      playAgain: 'Rejouer',
      changeSetup: 'Changer les reglages',
      summaryRobbers: (time) => `Braqueurs: ${time}`,
      summaryBank: (time) => `Banque: ${time}`,
      summaryRounds: (rounds) => `${rounds} manches`,
      turnLabel: (round, team) => `Manche ${round}, ${team}`,
    },
  };

  let screen = 'setup';
  let language = 'en';
  let robbers = 2;
  let secondsPerPlayer = 30;
  let round = 1;
  let turn = 'robbers';
  let running = false;
  let advancing = false;
  let remaining = 0;
  let timerId;
  let advanceTimeoutId;
  let countdownBeepTimeoutId;
  let lastSignalSecond = null;
  let audioContext;

  $: activePlayers = turn === 'robbers' ? robbers : BANK_EMPLOYEES;
  $: t = COPY[language];
  $: teamLabel = turn === 'robbers' ? t.robberTeam : t.bankTeam;
  $: turnDuration = activePlayers * secondsPerPlayer;
  $: progress = turnDuration > 0 ? ((turnDuration - remaining) / turnDuration) * 100 : 0;
  $: turnLabel = t.turnLabel(round, teamLabel.toLowerCase());

  function formatTime(value) {
    const minutes = Math.floor(value / 60);
    const seconds = value % 60;
    return `${minutes}:${String(seconds).padStart(2, '0')}`;
  }

  async function getAudioContext() {
    if (!audioContext) {
      const AudioContextClass = window.AudioContext || window.webkitAudioContext;
      audioContext = new AudioContextClass();
    }
    if (audioContext.state === 'suspended') {
      await audioContext.resume();
    }
    return audioContext;
  }

  async function playTone({ frequency = 660, duration = 0.08, type = 'square', volume = 0.16 }) {
    const context = await getAudioContext();
    const oscillator = context.createOscillator();
    const gain = context.createGain();

    oscillator.type = type;
    oscillator.frequency.value = frequency;
    gain.gain.setValueAtTime(volume, context.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.001, context.currentTime + duration);

    oscillator.connect(gain);
    gain.connect(context.destination);
    oscillator.start();
    oscillator.stop(context.currentTime + duration);
  }

  function playIntervalTick() {
    playTone({ frequency: 520, duration: 0.11, type: 'triangle', volume: 0.2 });
  }

  function playCountdownTick() {
    playTone({ frequency: 860, duration: 0.045, type: 'square', volume: 0.12 });
  }

  function playBell() {
    playTone({ frequency: 784, duration: 0.22, type: 'sine', volume: 0.24 });
    window.setTimeout(() => playTone({ frequency: 1046, duration: 0.24, type: 'sine', volume: 0.22 }), 150);
  }

  function playStartTone() {
    playTone({ frequency: 392, duration: 0.08, type: 'triangle', volume: 0.16 });
  }

  function clearCountdownBeeps() {
    window.clearTimeout(countdownBeepTimeoutId);
  }

  function countdownBeepDelay() {
    const secondsLeft = Math.max(0, Math.min(30, remaining));
    const urgency = (30 - secondsLeft) / 30;
    return Math.max(170, 1050 - urgency * 880);
  }

  function scheduleCountdownBeeps() {
    clearCountdownBeeps();

    if (!running || advancing) return;

    if (remaining > 30) {
      countdownBeepTimeoutId = window.setTimeout(scheduleCountdownBeeps, (remaining - 30) * 1000);
      return;
    }

    if (remaining > 0) {
      playCountdownTick();
      countdownBeepTimeoutId = window.setTimeout(scheduleCountdownBeeps, countdownBeepDelay());
    }
  }

  function startGame() {
    round = 1;
    turn = 'robbers';
    remaining = robbers * secondsPerPlayer;
    running = false;
    advancing = false;
    window.clearTimeout(advanceTimeoutId);
    clearCountdownBeeps();
    lastSignalSecond = null;
    screen = 'game';
  }

  async function startTurn() {
    if (advancing) return;
    await getAudioContext();
    playStartTone();
    running = true;
    lastSignalSecond = remaining;
    window.clearInterval(timerId);
    timerId = window.setInterval(tick, 1000);
    scheduleCountdownBeeps();
  }

  function stopTurn() {
    if (advancing) return;
    window.clearInterval(timerId);
    clearCountdownBeeps();
    running = false;
    advanceTurn();
  }

  function toggleTimer() {
    if (running) {
      stopTurn();
    } else {
      startTurn();
    }
  }

  function tick() {
    remaining -= 1;

    if (remaining <= 0) {
      window.clearInterval(timerId);
      clearCountdownBeeps();
      running = false;
      advancing = true;
      remaining = 0;
      playBell();
      advanceTimeoutId = window.setTimeout(advanceTurn, 750);
      return;
    }

    const elapsed = turnDuration - remaining;
    const isPlayerBoundary = elapsed > 0 && elapsed < turnDuration && elapsed % secondsPerPlayer === 0;
    const isFinalCountdown = remaining <= 30;

    if (!isFinalCountdown && isPlayerBoundary && lastSignalSecond !== remaining) {
      playIntervalTick();
    }

    lastSignalSecond = remaining;
  }

  function advanceTurn() {
    window.clearInterval(timerId);
    window.clearTimeout(advanceTimeoutId);
    clearCountdownBeeps();
    running = false;
    advancing = false;
    lastSignalSecond = null;

    if (turn === 'robbers') {
      turn = 'bank';
      remaining = BANK_EMPLOYEES * secondsPerPlayer;
      return;
    }

    if (round >= TOTAL_ROUNDS) {
      screen = 'done';
      remaining = 0;
      return;
    }

    round += 1;
    turn = 'robbers';
    remaining = robbers * secondsPerPlayer;
  }

  function resetToSetup() {
    window.clearInterval(timerId);
    window.clearTimeout(advanceTimeoutId);
    clearCountdownBeeps();
    running = false;
    advancing = false;
    screen = 'setup';
  }

  function replayGame() {
    startGame();
  }
</script>

{#if screen === 'setup'}
  <main class="setup-shell">
    <section class="setup-panel" aria-labelledby="setup-title">
      <div class="brand-mark" aria-hidden="true">$</div>
      <div class="title-block">
        <p class="eyebrow">{t.appEyebrow}</p>
        <h1 id="setup-title">{t.title}</h1>
        <p class="lede">
          {t.lede}
        </p>
      </div>

      <div class="controls-grid">
        <fieldset class="wide-field">
          <legend>{t.language}</legend>
          <div class="segmented" style={`--count: ${LANGUAGES.length}`}>
            {#each LANGUAGES as option}
              <button
                type="button"
                class:active={language === option}
                aria-pressed={language === option}
                on:click={() => (language = option)}
              >
                {LANGUAGE_LABELS[option]}
              </button>
            {/each}
          </div>
        </fieldset>

        <fieldset>
          <legend>{t.robbers}</legend>
          <div class="segmented" style={`--count: ${ROBBER_OPTIONS.length}`}>
            {#each ROBBER_OPTIONS as option}
              <button
                type="button"
                class:active={robbers === option}
                aria-pressed={robbers === option}
                on:click={() => (robbers = option)}
              >
                {option}
              </button>
            {/each}
          </div>
          <p class="hint">{t.plusBank}</p>
        </fieldset>

        <fieldset>
          <legend>{t.timePerPlayer}</legend>
          <div class="segmented" style={`--count: ${TIME_OPTIONS.length}`}>
            {#each TIME_OPTIONS as option}
              <button
                type="button"
                class:active={secondsPerPlayer === option}
                aria-pressed={secondsPerPlayer === option}
                on:click={() => (secondsPerPlayer = option)}
              >
                {option}s
              </button>
            {/each}
          </div>
        </fieldset>

      </div>

      <div class="summary-strip">
        <span>{t.summaryRobbers(formatTime(robbers * secondsPerPlayer))}</span>
        <span>{t.summaryBank(formatTime(BANK_EMPLOYEES * secondsPerPlayer))}</span>
        <span>{t.summaryRounds(TOTAL_ROUNDS)}</span>
      </div>

      <button class="primary-action" type="button" on:click={startGame}>
        {t.startGame}
      </button>
    </section>
  </main>
{:else if screen === 'game'}
  <main class:running class="game-shell">
    <section class="status-bar" aria-label="Game status">
      <button type="button" class="icon-button" title={t.backToSetup} aria-label={t.backToSetup} on:click={resetToSetup}>
        <span aria-hidden="true">&larr;</span>
      </button>
      <div>
        <p>{t.round}</p>
        <strong>{round} / {TOTAL_ROUNDS}</strong>
      </div>
      <div>
        <p>{t.turn}</p>
        <strong>{teamLabel}</strong>
      </div>
      <div>
        <p>{t.players}</p>
        <strong>{activePlayers}</strong>
      </div>
    </section>

    <section class="timer-stage" aria-labelledby="turn-title">
      <div class="turn-copy">
        <p class="eyebrow">{turnLabel}</p>
        <h1 id="turn-title">{teamLabel}</h1>
      </div>

      <div class="timer-face" aria-live="polite">
        <div class="progress-ring" style={`--progress: ${progress}%`}>
          <span>{formatTime(remaining)}</span>
        </div>
      </div>

      <button class="timer-button" class:stop={running} type="button" on:click={toggleTimer} disabled={advancing}>
        {advancing ? t.done : running ? t.stop : t.start}
      </button>
    </section>
  </main>
{:else}
  <main class="done-shell">
    <section class="done-panel">
      <div class="brand-mark" aria-hidden="true">$</div>
      <p class="eyebrow">{t.finalBell}</p>
      <h1>{t.gameComplete}</h1>
      <p class="lede">{t.allRoundsFinished(TOTAL_ROUNDS)}</p>
      <div class="done-actions">
        <button class="primary-action" type="button" on:click={replayGame}>{t.playAgain}</button>
        <button class="secondary-action" type="button" on:click={resetToSetup}>{t.changeSetup}</button>
      </div>
    </section>
  </main>
{/if}
