<style>@import url('../css/gadgets/termynal.css');</style>

<style>
    body {
        
    }
</style>

<div id="termynal"></div>

<script src="../js/gadgets/termynal.js" data-termynal-container="#termynal"></script>
<script>
    var termynal = new Termynal('#termynal', {
    typeDelay: 40,
    lineDelay: 700,
    lineData: [
        { type: 'input', prompt: '$', value: 'show content' },
        { value: 'Sure?' },
        { type: 'input',  typeDelay: 1000, prompt: '(y/n)', value: 'y' },
        { type: 'progress', progressChar: '.' },
        { value: 'Notes of Computer System' },
    ]
    });
</script>