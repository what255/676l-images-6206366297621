## 676l Payment Page Images

Source page:

`https://676l.com/sc/zhanghao/6206366297621/0.18`

This repository is organized so humans and other AI tools can understand the image purpose without reverse-engineering the page script.

### Directory layout

- `images/wallet-icons/`
  Wallet brand icons shown on the payment page.
- `images/manual-payment/`
  Manual payment tutorial screenshots, grouped by wallet.

### Wallet icons

- `images/wallet-icons/imtoken.png`
- `images/wallet-icons/tokenpocket.png`
- `images/wallet-icons/okx-web3-wallet.png`
- `images/wallet-icons/metamask.png`
- `images/wallet-icons/trust-wallet.png`
- `images/wallet-icons/bitkeep.png`
- `images/wallet-icons/bitpie.jpg`
- `images/wallet-icons/tronlink.png`

### Manual payment tutorials

Each wallet folder contains:

- `step-1.png`
- `step-2.png`

Wallet folders:

- `images/manual-payment/bitpie/`
- `images/manual-payment/imtoken/`
- `images/manual-payment/tokenpocket/`
- `images/manual-payment/okx-web3-wallet/`
- `images/manual-payment/tronlink/`
- `images/manual-payment/metamask/`
- `images/manual-payment/trust-wallet/`
- `images/manual-payment/bitkeep/`

### Source mapping from page script

- `bitpie`: originally `15.png`, `16.png`
- `imtoken`: originally `1.png`, `2.png`
- `tokenpocket`: originally `3.png`, `4.png`
- `okx-web3-wallet`: originally `5.png`, `6.png`
- `tronlink`: originally `7.png`, `8.png`
- `metamask`: originally `9.png`, `10.png`
- `trust-wallet`: originally `11.png`, `12.png`
- `bitkeep`: originally `13.png`, `14.png`

### Notes

- The original page reuses some wallet icons in multiple places; only unique icon files are stored here.
- The page also includes inline SVG elements, but those are embedded in HTML and were not separate downloadable image files.
