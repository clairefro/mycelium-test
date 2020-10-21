# Mycelium

Treat this repo like the hidden web of vegetative fungus under the forest floor - you don't interact with it but it sprouts translated content to the surface.

Less poetically, don't edit the code in this repo directly. Make all translation pull requests through the [gitlocalize dashboard for this repo](https://gitlocalize.com/repo/5446).

The idea would be to sync the main English repo to the `redwoodjs.com` directory, which serves as the source directory for translations. Tranlsations for each locale are set up to make auto-prs into a mirrored doc structure, under their domain within the `i18n` dir.

Then these completed translations would be auto-PRed to their respective subdomain repos (ex: `fr.redwood.js`) to overwrite the existing English default md files.

Maybe too complicated
