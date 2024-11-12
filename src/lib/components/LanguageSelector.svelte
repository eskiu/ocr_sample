<script lang="ts">
  export let selectedLanguages: string[] = ['eng', 'spa'];
  export let onLanguagesChanged: (data: { selectedLanguages: string[] }) => void = () => {};

  interface LanguageOption {
    code: string;
    name: string;
  }

  const availableLanguages: LanguageOption[] = [
    { code: 'eng', name: 'Inglés' },
    { code: 'spa', name: 'Español' },
  ];

  function handleSelectionChange() {
    onLanguagesChanged({ selectedLanguages });
  }
</script>

<div class="flex gap-4 mx-auto">
  <h3>Selecciona Idiomas:</h3>
  {#each availableLanguages as lang}
    <label>
      <input
        type="checkbox"
        value="{lang.code}"
        bind:group="{selectedLanguages}"
        on:change="{handleSelectionChange}"
      />
      {lang.name}
    </label>
  {/each}
</div>
