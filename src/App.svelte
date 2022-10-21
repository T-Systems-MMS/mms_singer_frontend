<script>
  import { onMount } from "svelte";
  import RangeSlider from "svelte-range-slider-pips";
  import { Wave, Circle2 } from 'svelte-loading-spinners'

  let folders_check = []
  let midiFileInput
  let upload = {
    active: false,
    textnotegenActive: false,
    name: '',
    phonemes: '',
    midiFile: null,
  }
  let selected = {
    active: false,
    folder: '',
    lyrics: '',
    text: '',
    melody: '',
    all: false,
    all_index: 0,
    folders: []
  }
  let config = {
    active: false,
    d_config: 1.0,
    pv_config: 1.0,
    i_config: 1.0,
    h_config: 1.0,
    shift_config: 0.0,
    gsts: [],
    choir_mode: 0,
    choir_mode_variance: 0.5,
    split_method: "auto",
    guidance: "syllable",
    vocoder: "hifi_gan",
    use_diff: true,
    split_long_notes: 20.0,
    pad_nosounds: 0.0,
  }
  let results = {
    active: false,
    waiting: false,
    refreshing_folders: false,
    wav_files: [],
    images: [],
    synth_all: false
  }
  let hparams = {
    use_gst: false,
    gst_token_num: 4
  }

  onMount(async () => {
    getFolders()
    const res = await fetch('/api/hparams', {method: 'GET'})
    const json = await res.json()
    hparams['use_gst'] = json['use_gst']
    hparams['gst_token_num'] = json['gst_token_num']
    if(hparams['use_gst']) {
      config['gsts'] = Array(hparams['gst_token_num']).fill(0)
    }
  })

  function toggleUpload() {
    upload['active'] = !upload['active']  
  }

  async function onMidiSelected(e) {
    let reader = new FileReader();
    reader.readAsDataURL(e.target.files[0]);
    reader.onload = e => {
      const regex = /data:.*base64,/
      upload.midiFile = e.target.result.replace(regex, "");
      console.log(upload.midiFile);
    }
  }

  async function performUpload() {
    await fetch('/api/upload', {
      method: 'POST',
      body: JSON.stringify({
        "text": upload.phonemes,
        "melody": upload.midiFile,
        "name": upload.name,
        "lyrics": upload.phonemes,
      }),
      headers: {
        "content-type": "application/json"
      }
    })

    upload['active'] = false;
    upload['phonemes'] = '';
    upload['name'] = '';
    upload['midiFile'] = null;
    await getFolders();
  }

  async function getFolders() {
    results.refreshing_folders = true
    const res = await fetch('/api/data', {method: 'GET'})
    const json = await res.json()
    folders_check = []
    for(let entry in json) {
      folders_check.push({
        name: json[entry].name, 
        melody: json[entry].melody,
        lyrics: json[entry].lyrics,
        checked: false,
        wav_exists: false,
        wavs: []
      })
    }
    results.refreshing_folders = false
  }

  async function deleteFolder(folder) {
    await fetch('api/data?folder=' + folder, {method: 'DELETE'})
    await getFolders()
  }

  async function deleteSelected() {
    let checked = folders_check.filter(t => t.checked)
    for(let folder of checked) {
      deleteFolder(folder.name)
    }
  }

  async function selectAll() {
    for(let folder of folders_check) {
      folder.checked = true
    }
    folders_check = [...folders_check]
  }   

  async function selectNone() {
    for(let folder of folders_check) {
      folder.checked = false
    }
    folders_check = [...folders_check]
  }  

  async function invokeTextnotegen() {
    upload['textnotegenActive'] = true;
    await fetch('/api/textnotegen', {method: 'POST'})
    upload['textnotegenActive'] = false;
    await getFolders();
  }

  async function synthesize() {
    results['active'] = true
    results['waiting'] = true
    
    try {
      const res = await fetch('/api/synthesize', {
        method: 'POST',
        body: JSON.stringify({
          "file_name": selected['folder'],
          "text": selected['text'],
          "melody": selected['melody'],
          "d_config": config['d_config'],
          "pv_config": config['pv_config'],
          "i_config": config['i_config'],
          "h_config": config['h_config'],
          "choir_mode": config['choir_mode'],
          "choir_mode_variance": config['choir_mode_variance'],
          "style_token_weights": config['gsts'],
          "split_method": config['split_method'],
          "guidance": config['guidance'],
          "vocoder": config['vocoder'],
          "use_diff": config['use_diff'],
          "shift_config": config['shift_config'],
          "pad_nosounds": config['pad_nosounds'],
          "split_long_notes": config['split_long_notes'],
        }),
        headers: {
          "content-type": "application/json"
        }
      })
    
      if(res.status != 200) {
        const err = await res.text()
        console.log(err)
        results['waiting'] = false
        return []
      }
      const json = await res.json()
      const wavs = json['wav_files']
      results['waiting'] = false
      results['wav_files'] = results['wav_files'].concat(wavs)
      results['images'] = json['images']
      return wavs
    } catch (error) {
      console.log(error)
      results['waiting'] = false
      return []
    }
  }

  async function synthesize_checked() {
    results.synth_all = true
    results.active = true
    results.wav_files = []
    let checked_indices = Array.from(folders_check.entries()).filter(([idx, folder]) => folder.checked).map(([idx, folder]) => idx)
    for(let idx of checked_indices) {
      if(results.synth_all) {
        selected.folder = folders_check[idx].name
        selected.text = folders_check[idx].lyrics
        selected.melody = folders_check[idx].melody
        folders_check[idx].wavs = []
        folders_check[idx].wav_exists = false
        folders_check[idx].wavs = await synthesize()
        folders_check[idx].wav_exists = true
      }
    }
  }
</script>

<main>
  <div class="row">
    <div class="column left">
      <h2 class="magenta">Data Selection</h2>
      <center>
        <button class="button" type="button" on:click={getFolders}>Refresh folders</button>
        <button class="button" type="button" on:click={invokeTextnotegen}>Invoke Textnotegen</button>
        <button class="button" type="button" on:click={toggleUpload}>Toggle upload</button>

        {#if upload['textnotegenActive']}
          <p>Textnotegen is doing magic</p>
        {/if}

        {#if upload['active']}
          <div><input type="text" bind:value={upload['name']} placeholder="Name" /></div>
          <div><textarea bind:value={upload['phonemes']} placeholder="Phonemes" /></div>
          <div><input type="file" accept=".mid" on:change={(e)=>onMidiSelected(e)} bind:this={midiFileInput} /></div>
          
          {#if upload['midiFile']}
            <div><button class="button" type="button" on:click={performUpload}>Upload</button></div>
          {/if}
        {/if}
      </center>

      {#if upload.textnotegenActive | results['refreshing_folders']}
        <center style="margin-top:40px;margin-bottom:40px">
          <div class="spinner-item">
            <Circle2 colorOuter="#E20074"/>
          </div>
        </center>
      {/if} 

      <ul style="list-style-type:none">
        <button class="button" style="margin-bottom:10px;" type="button" on:click={selectAll}>Select All</button>
        <button class="button" style="margin-bottom:10px;" type="button" on:click={selectNone}>Unselect</button>
        {#each folders_check as folder}
          <div class:done={folder.checked}> 
	          <input type=checkbox bind:checked={folder.checked}>
            {folder.name}
          </div>
        {/each}
        <button class="button" style="margin_top:20px;" type="button" on:click={deleteSelected}>Delete Selected</button>
      </ul>
    </div>
      
    <div class="column middle">
      <h2 class="magenta">Synthesis</h2>
      {#if folders_check.filter(f => f.checked) === []}
        Select the files you want to synthesize.
      {:else}
        <button class="button" style="margin_bottom:10px;" type="button" on:click={synthesize_checked}>Synthesize</button>
        <button class="button" style="margin_bottom:10px;" type="button" on:click={results.synth_all = false}>Interrupt</button>
      {/if}

      {#each folders_check.filter(f => f.checked) as folder}
        <div style="margin-top:20px;">{folder.name}</div>
        <div>{folder.lyrics}</div>
        {#if folder.wav_exists}
          {#each folder.wavs as wav}
            <figure style="margin-bottom:20px;margin-top:10px;">
              <audio controls title={wav.filename} id="audio_{wav.filename}" bind>
                <source src="data:audio/wav;base64,{wav.data}">
              </audio>
            </figure>
          {/each}
        {/if}
      {/each}

      {#if results['waiting']}
        <center style="margin-top:40px;">
          <div class="spinner-item">
            <Wave color="#E20074"/>
          </div>
        </center>
      {/if} 
    </div>

    <div class="column right">
      <h2 class="magenta">Configuration</h2>
        Pitch shift: {config['shift_config']}
        <RangeSlider values={[config['shift_config']]} min={-20} max={20} step={1} pipstep={50}  first="label" last="label" pips on:change={(e) => {config.shift_config = e.detail.value}} />
    <!--   
        Timestretch: {config['d_config']}
        <RangeSlider values={[config['d_config']]} min={0} max={2} step={0.01} pipstep={50}  first="label" last="label" pips on:change={(e) => {config.d_config = e.detail.value}} />   
        Pitch variance: {config['pv_config']}
        <RangeSlider values={[config['pv_config']]} min={-1} max={2} step={0.01} pipstep={50}  first="label" last="label" pips on:change={(e) => {config.pv_config = e.detail.value}} />
        Intensity: {config['i_config']}
        <RangeSlider values={[config['i_config']]} min={0} max={2} step={0.01} pipstep={50}  first="label" last="label" pips on:change={(e) => {config.i_config = e.detail.value}} />
        Harmonicity: {config['h_config']}
        <RangeSlider values={[config['h_config']]} min={0} max={2} step={0.01} pipstep={50}  first="label" last="label" pips on:change={(e) => {config.h_config = e.detail.value}} />
    -->    
        {#each config['gsts'] as gst, i}
          GST {i}: {gst}
          <RangeSlider values={[gst]} min={-0.3} max={1.7} step={0.01} pipstep={50}  first="label" last="label" pips on:change={(e) => {config['gsts'][i] = e.detail.value}} />
        {/each}
        Choir Mode: {config['choir_mode']}
        <RangeSlider values={[config['choir_mode']]} min={0} max={20} step={1} pips first="label" last="label" on:change={(e) => {config['choir_mode'] = e.detail.value;}} />
        Choir Mode Variance: {config['choir_mode_variance']}
        <RangeSlider values={[config['choir_mode_variance']]} min={0} max={1} step={0.01} pipstep={50} first="label" last="label" on:change={(e) => {config['choir_mode_variance'] = e.detail.value;}} />
        Pad nosounds to snippet ends: {config['pad_nosounds']}s
        <RangeSlider values={[config['pad_nosounds']]} min={0} max={1} step={0.01} pipstep={50} first="label" last="label" on:change={(e) => {config['pad_nosounds'] = e.detail.value;}} />
        Split notes longer than: {config['split_long_notes']}s
        <RangeSlider values={[config['split_long_notes']]} min={1} max={20} step={0.1} pipstep={50} first="label" last="label" on:change={(e) => {config['split_long_notes'] = e.detail.value;}} />        
        <div>
          Split Method: 
          <select bind:value={config['split_method']}>
            <option value={"none"}>None</option>
            <option value={"auto"}>Auto</option>
            <option value={"all"}>All</option>
          </select>
        </div>
        <div>
          Guidance:
          <select bind:value={config['guidance']}>
            <option value={"none"}>None</option>
            <option value={"syllable"}>Syllable</option>
          </select>
        </div>
        <div>
          Decoder:
          <select bind:value={config['use_diff']}>
            <option value={false}>Transformer</option>
            <option value={true}>Diffusion</option>
          </select>
        </div>
        <div>
          Vocoder:
          <select bind:value={config['vocoder']}>
            <option value={"hifi_gan"}>Hifi-Gan</option>
            <option value={"hifi_gan_new"}>Hifi-Gan New</option>
            <option value={"griffin_lim"}>Griffin-Lim</option>
          </select>
        </div>
    </div>
  </div>
</main>

<style>
  :root {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
      Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  }

  main {
    padding: 1em;
    margin: 0 auto;
  }
  h2.magenta {
    text-align: center;
    color: #E20074;
  }

  .column {
    float: left;
    margin: 3%;
  }

  .left {
    width: 25%;
    text-align: left;
  }

  .right {
    width: 25%;
    text-align: center;
  }

  .middle {
    text-align: center;
    width: 32%;
  }
  @media screen and (max-width: 1500px) {
    .column {
      width: 94%;
    }
  }
</style>
