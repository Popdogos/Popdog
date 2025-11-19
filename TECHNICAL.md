# Technical Documentation

## Architecture Overview

Popdog is built as a single-page application using vanilla web technologies. No build tools or frameworks are required, making it lightweight and fast.

## Technology Stack

### Frontend
- **HTML5**: Semantic markup with proper structure
- **CSS3**: Modern styling with:
  - CSS Variables for theming
  - Flexbox and Grid for layouts
  - CSS Animations and Transitions
  - Media Queries for responsiveness
- **Vanilla JavaScript (ES6+)**: 
  - No dependencies
  - Event-driven architecture
  - Async/await for wallet operations

### Blockchain Integration
- **Solana Web3.js**: Via browser wallet extensions
- **Wallet Adapter**: Phantom and Solflare support
- **Network**: Solana Mainnet

## File Structure

```
popdog/
├── index.html              # Main application file
├── styles.css              # All styles (1615+ lines)
├── Assets/
│   ├── logo.jpg
│   ├── banner.jpg
│   ├── mouthopen.jpg
│   ├── mouthclose.png
│   ├── meme.jpg
│   ├── pic.jpg
│   └── cat-mouth-noise-192-kbps.mp3
└── Documentation/
    ├── README.md
    ├── CONTRIBUTING.md
    └── TECHNICAL.md
```

## Key Features Implementation

### 1. Interactive Popdog

**Location**: Hero section

**Implementation**:
- Image toggle between `mouthopen.jpg` and `mouthclose.png`
- Click event listener on `#popdogImage`
- CSS animations: `popdogBounce`, `mouthOpen`, `mouthClose`
- Audio playback using HTML5 Audio API
- Counter increment with localStorage persistence (optional)

**Code Pattern**:
```javascript
popdogImage.addEventListener('click', function() {
    playPopSound();
    toggleMouthState();
    incrementCounter();
    triggerAnimations();
});
```

### 2. Send PopDog Transfer Panel

**Location**: Send PopDog section

**Implementation**:
- Real-time USD conversion using `POPDOG_PRICE` constant
- Input validation and formatting
- Responsive layout with flexbox
- Mobile-optimized touch targets (44px minimum)

**Key Functions**:
- `updateSendPopdogUSDValue()`: Calculates USD equivalent
- `checkSendPopdogWallet()`: Detects wallet connection
- `connectSendPopdogWallet()`: Handles wallet connect/disconnect

### 3. Wallet Integration

**Implementation**:
- Detects `window.solana` (Phantom) and `window.solflare`
- Uses Solana Wallet Adapter API
- Event listeners for `connect` and `disconnect` events
- Address truncation for display (first 4 + last 4 chars)

**Error Handling**:
- Try-catch blocks for connection errors
- User-friendly error messages
- Fallback for missing wallet extensions

### 4. Responsive Design

**Breakpoints**:
- Desktop: > 768px
- Tablet: 481px - 768px
- Mobile: ≤ 480px

**Mobile Optimizations**:
- Touch-friendly targets (44px minimum)
- iOS zoom prevention (16px font-size minimum)
- Flexible layouts with CSS Grid and Flexbox
- Optimized font sizes and spacing

**CSS Variables**:
```css
:root {
    --bg-yellow: #FFF8E1;
    --card-bg: #FFFFFF;
    --accent-yellow: #FFD700;
    --text-main: #2C2C2C;
    --text-muted: #666666;
}
```

## Performance Optimizations

1. **No External Dependencies**: Zero npm packages, faster load times
2. **CSS Optimization**: 
   - Efficient selectors
   - Hardware-accelerated animations
   - Minimal repaints/reflows
3. **JavaScript Optimization**:
   - Event delegation where possible
   - Debounced input handlers
   - Lazy loading for images (optional)
4. **Mobile Performance**:
   - Touch-action CSS properties
   - Reduced animation complexity on mobile
   - Optimized image sizes

## Browser Compatibility

### Supported Browsers
- Chrome 90+
- Firefox 88+
- Safari 14+
- Edge 90+
- Mobile browsers (iOS Safari, Chrome Mobile)

### Polyfills
None required - uses standard web APIs only.

## Security Considerations

1. **Wallet Security**:
   - No private key storage
   - All wallet operations via browser extension
   - User must explicitly approve transactions

2. **Input Validation**:
   - Amount input validation
   - Address format checking (basic)
   - XSS prevention via proper HTML escaping

3. **Content Security**:
   - No inline scripts (except necessary)
   - External resources loaded securely
   - HTTPS recommended for production

## Accessibility

- Semantic HTML5 elements
- Proper heading hierarchy
- Alt text for images
- Keyboard navigation support
- Screen reader friendly
- ARIA labels where needed

## x402 Payment System

The project includes comprehensive x402 payment system integration for accepting PopDog token payments.

### Documentation Files

- **X402_INTEGRATION.md**: Complete integration guide covering:
  - x402 protocol overview
  - SDK options (Rapid402, xPlug, Coinbase reference)
  - Implementation steps
  - API reference
  - Security considerations
  - Best practices

- **X402_SDK_GUIDE.md**: Quick start guide with:
  - Installation instructions
  - Basic setup and configuration
  - Code examples
  - Error handling
  - Testing guide
  - Production checklist

### Current Implementation

The x402 payment card is implemented in the hero section with:
- Wallet connection detection
- Payment amount display
- Network information
- Payment button handler

### SDK Integration

Recommended SDK: **Rapid402 SDK** for Solana
- GitHub: https://github.com/rapid402/rapid402-sdk
- TypeScript/JavaScript support
- Solana blockchain optimized
- Easy integration

See the x402 documentation files for complete implementation details.

## Future Enhancements

Potential improvements:
1. Full x402 SDK integration (Rapid402)
2. Server-side payment verification
3. Payment history tracking
4. Service Worker for offline support
5. Progressive Web App (PWA) capabilities
6. Dark mode toggle
7. Internationalization (i18n)
8. Advanced wallet features (transaction signing)
9. Real-time price updates via API
10. Analytics integration
11. Error logging service

## Deployment

### Static Hosting
The site can be deployed to any static hosting service:
- GitHub Pages
- Netlify
- Vercel
- Cloudflare Pages
- AWS S3 + CloudFront

### Requirements
- HTTPS (required for wallet connections)
- Proper CORS headers
- GZIP compression
- Cache headers for assets

## Testing

### Manual Testing Checklist
- [ ] Wallet connection (Phantom)
- [ ] Wallet connection (Solflare)
- [ ] Wallet disconnection
- [ ] Amount input and USD conversion
- [ ] Popdog interaction
- [ ] Responsive design (mobile, tablet, desktop)
- [ ] Cross-browser compatibility
- [ ] Touch interactions on mobile
- [ ] iOS zoom prevention
- [ ] Error handling

## Maintenance

### Regular Updates
- Keep wallet adapter APIs updated
- Monitor browser compatibility
- Update dependencies (if any added)
- Security patches
- Performance monitoring

## Support

For technical issues or questions:
- Open an issue on GitHub
- Check existing documentation
- Contact maintainers

---

Last updated: January 2025

