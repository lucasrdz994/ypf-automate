<script>
  import {
    Header,
    Content,
    Grid,
    Row,
    Column,
    TextArea,
    Button,
    Checkbox,
    InlineNotification,
    ToastNotification,
    Modal,
    DataTable
  } from 'carbon-components-svelte'

  import LogoSvelte from 'carbon-icons-svelte/lib/LogoSvelte.svelte'
  import Favorite from 'carbon-icons-svelte/lib/Favorite.svelte'

  import { onMount } from 'svelte'

  // Estructura de table
  const tableHeaders = [
    { key: 'envelope', value: 'Sobre' },
    { key: 'amount', value: 'Importe' }
  ]

  let inBulk = false
  let feedbackModal = false
  let errorNotification = false
  let exportedMapped = ''
  let receivedMapped = ''
  let inputEmpty = ''
  let diffEmpty = ''

  let input = {
    exported: {},
    received: {}
  }

  let diff = {
    exported: {},
    received: {},
    amount: {},
    inBulk: 0
  }

  $: exportedMissing = Boolean(Object.keys(diff.exported).length)
  $: receivedMissing = Boolean(Object.keys(diff.received).length)
  $: amountDiff = Boolean(Object.keys(diff.amount).length)
  $: anyDiff = exportedMissing || receivedMissing || amountDiff

  $: tableRows = {
    exported: Object.entries(diff.exported).map((el) => {
      return {
        id: el[0],
        envelope: el[0],
        amount: `$ ${el[1]}`
      }
    }),
    received: Object.entries(diff.received).map((el) => {
      return {
        id: el[0],
        envelope: el[0],
        amount: `$ ${el[1]}`
      }
    }),
    amount: Object.entries(diff.amount).map((el) => {
      return {
        id: el[0],
        envelope: el[0],
        amount: `$ ${el[1]}`
      }
    })
  }

  $: amountDiffTotal = Object.values(diff.amount).reduce((acc, curr) => (acc += curr), 0)

  async function handlePaste(event) {
    event.preventDefault()
    const inputName = event.target.name
    const inputPaste = event.clipboardData.getData('text')
    let rows = inputPaste.split(/\r?\n|\r|\n/g)

    if (inputName === 'exported') {
      input.exported = {}
      exportedMapped = ''
    }
    if (inputName === 'received') {
      input.received = {}
      receivedMapped = ''
    }
    try {
      for (let [index, row] of rows.entries()) {
        row = row.trim()
        let [envelope, amount] = row.split(/\s+/g)
        envelope = String(envelope)
        amount = amount.replace(',', '.')
        amount = Number(amount)

        if (isNaN(amount)) throw new Error('error')

        const text = `${envelope} ($${amount}) ${rows[index + 1] ? ', ' : ''}`
        if (inputName === 'exported') {
          input.exported = { ...input.exported, [envelope]: amount }
          exportedMapped += text
        }
        if (inputName === 'received') {
          input.received = { ...input.received, [envelope]: amount }
          receivedMapped += text
        }
      }
    } catch (error) {
      errorNotification = true
      if (inputName === 'exported') {
        input.exported = {}
        exportedMapped = ''
      }
      if (inputName === 'received') {
        input.received = {}
        receivedMapped = ''
      }
    }
  }

  async function handleCalculate() {
    if (Object.keys(input.exported).length === 0 || Object.keys(input.received).length === 0) return
    diff = JSON.parse(diffEmpty)
    for (const envelope in input.exported) {
      // A Granel
      if (inBulk) {
        if (Number(envelope) < 1000) {
          diff.inBulk -= input.exported[envelope]
          continue
        }
      }
      // Si no encuentro el sobre, no lo sumo en el cálcul
      if (!input.received.hasOwnProperty(envelope)) {
        diff.received[envelope] = input.exported[envelope]
        continue
      }

      if (input.exported[envelope] !== input.received[envelope]) {
        diff.amount[envelope] = input.received[envelope] - input.exported[envelope]
      }
    }

    for (const envelope in input.received) {
      // A Granel
      if (inBulk) {
        if (Number(envelope) < 1000) {
          diff.inBulk += input.received[envelope]
          continue
        }
      }
      if (!Object.keys(input.exported).includes(envelope)) {
        diff.exported[envelope] = input.received[envelope]
      }
    }
    feedbackModal = true
  }

  function handleReset() {
    exportedMapped = ''
    receivedMapped = ''
    input = JSON.parse(inputEmpty)
    diff = JSON.parse(diffEmpty)
  }

  function handleBackspace(event) {
    if (event.code === 'Backspace') {
      const inputName = event.target.name
      if (inputName === 'exported') {
        input.exported = {}
        exportedMapped = ''
      }
      if (inputName === 'received') {
        input.received = {}
        receivedMapped = ''
      }
    }
  }

  function handleDownload() {
    let rows = [
      [
        'Sobre',
        'Importe estación (exportado)',
        'Importe recaudadora (recibido por email)',
        'Diferencia de importes',
        'Estado'
      ]
    ]
    let items = []

    for (const envelope of Object.keys(input.received)) {
      let status = 'ok'
      let exportedAmount
      let amountDiff

      input.exported.hasOwnProperty(envelope)
        ? (exportedAmount = input.exported[envelope])
        : (status = 'Sobre no encontrado en la estación (exportado)')

      input.exported.hasOwnProperty(envelope)
        ? (amountDiff = input.received[envelope] - input.exported[envelope])
        : (amountDiff = 'N/A')

      items.push([envelope, exportedAmount, input.received[envelope], amountDiff, status])
    }

    for (const envelope of Object.keys(input.exported)) {
      if (Object.keys(input.received).includes(envelope)) continue
      let status = 'ok'
      let receivedAmount
      let amountDiff

      input.received.hasOwnProperty(envelope)
        ? (receivedAmount = input.exported[envelope])
        : (status = 'Sobre no encontrado en la recaudadora (recibido por email)')

      input.received.hasOwnProperty(envelope)
        ? (amountDiff = input.received[envelope] - input.exported[envelope])
        : (amountDiff = 'N/A')

      items.push([envelope, input.exported[envelope], receivedAmount, amountDiff, status])
    }

    items = items.sort((a, b) => {
      return Number(a.at(0)) - Number(b.at(0))
    })

    rows = rows.concat(items)

    let csvContent = 'data:text/csv;charset=utf-8,' + rows.map((e) => e.join(',')).join('\n')
    let encodedUri = encodeURI(csvContent)
    let link = document.createElement('a')
    link.style.visibility = 'hidden'
    link.setAttribute('href', encodedUri)
    link.setAttribute('download', Date.now() + '.csv')
    document.body.appendChild(link)
    link.click()
    link.remove()
  }

  onMount(() => {
    inputEmpty = JSON.stringify(input)
    diffEmpty = JSON.stringify(diff)
  })
</script>

<Header company="YPF" platformName="Automate" />
<Content>
  <Grid>
    <h1>Automatización</h1>

    <Checkbox labelText="Contiene a granel" bind:checked={inBulk} />

    <InlineNotification
      kind="info"
      title="Nota:"
      subtitle="Copiá y pegá las dos columnas del excel correspondientes a los sobres e importes, en ese orden."
    />

    <h4>Estación (exportado)</h4>
    <Row padding>
      <Column>
        <TextArea
          labelText="Datos"
          placeholder="Pegá los datos acá..."
          name="exported"
          readonly
          bind:value={exportedMapped}
          on:paste={handlePaste}
          on:keydown={handleBackspace}
        />
      </Column>
    </Row>
    <h4>Recaudadora (recibido por email)</h4>
    <Row padding>
      <Column>
        <TextArea
          labelText="Datos"
          placeholder="Pegá los datos acá..."
          name="received"
          readonly
          bind:value={receivedMapped}
          on:paste={handlePaste}
          on:keydown={handleBackspace}
        />
      </Column>
    </Row>

    <Button kind="secondary" on:click={handleReset}>Reiniciar</Button>
    <Button on:click={handleCalculate}>Calcular</Button>

    <Modal
      bind:open={feedbackModal}
      hasScrollingContent
      modalHeading="Resultado"
      primaryButtonText="Guardar"
      secondaryButtonText="Cerrar"
      on:click:button--secondary={() => (feedbackModal = false)}
      on:click:button--primary={handleDownload}
    >
      {#if receivedMissing}
        <div class="feedback-modal-content">
          <h5>Sobres faltantes en la recaudadora (recibidos por email)</h5>
          <p>Son los sobres que se encuentran en el excel recibido por email pero no en el exportado.</p>
          <DataTable headers={tableHeaders} rows={tableRows.received} />
        </div>
      {/if}
      {#if exportedMissing}
        <div class="feedback-modal-content">
          <h5>Sobres faltantes en la estación (exportados)</h5>
          <p>Son los sobres que se encuentran en el excel recibido por email pero no en el exportado.</p>
          <DataTable headers={tableHeaders} rows={tableRows.exported} />
        </div>
      {/if}
      {#if amountDiff}
        <div class="feedback-modal-content">
          <h5>Diferencia de importes</h5>
          <p>Son los importes diferentes</p>
          <DataTable headers={tableHeaders} rows={tableRows.amount} />
          <p>Total: $ {amountDiffTotal}</p>
        </div>
      {/if}
      {#if inBulk}
        <h5>Diferencia de importes a granel</h5>
        <p>$ {diff.inBulk}</p>
      {/if}
      {#if !anyDiff}
        <p>No encontramos errores o diferencias en la consiciliación.</p>
      {/if}
    </Modal>
  </Grid>
</Content>
<footer>
  Hecho con <Favorite size={16} /> y <LogoSvelte size={16} /> por Lucas Rodríguez
</footer>

{#if errorNotification}
  <div class="error-notification">
    <ToastNotification
      title="Error"
      subtitle="Los datos ingresados no se encuentran en un formato correcto."
      caption={new Date().toLocaleString()}
      timeout={5000}
      on:close={() => (errorNotification = false)}
    />
  </div>
{/if}

<style>
  .feedback-modal-content {
    margin: 1rem 0;
  }
  .error-notification {
    position: fixed;
    top: 3rem;
    right: 0;
  }
  footer {
    height: 52px;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 1rem;
    background: #393939;
    width: 100%;
    color: #fff;
    gap: 5px;
  }
</style>
