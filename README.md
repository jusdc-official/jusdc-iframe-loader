/**
 * Advanced JUSDC iFrame Loader with Error Handling and Retry Logic
 * @param {string} iframeId - The ID of the iframe element
 * @param {string} url - The URL to load in the iframe
 * @param {Object} [options] - Configuration options
 * @param {number} [options.retryCount=3] - Number of retry attempts
 * @param {number} [options.retryDelay=2000] - Delay between retries in ms
 * @param {boolean} [options.showLoader=true] - Whether to show loading spinner
 * @param {boolean} [options.cacheBust=false] - Add cache-busting parameter
 */
function loadJUSDCIframe(iframeId, url, options = {}) {
    const config = {
        retryCount: options.retryCount || 3,
        retryDelay: options.retryDelay || 2000,
        showLoader: options.showLoader !== false,
        cacheBust: options.cacheBust || false,
        currentAttempt: 0
    };

    const iframe = document.getElementById(iframeId);
    if (!iframe) {
        console.error(`JUSDC Loader: Iframe with ID ${iframeId} not found`);
        return;
    }

    const loadingDiv = document.getElementById(`loading${iframeId.replace('iframe', 'frame')}`);
    const container = iframe.closest('.table-responsive') || iframe.parentElement;
    const messageSpan = document.getElementById(`ContentPlaceHolder1_${iframeId.replace('iframe', 'Message')}`);

    // Reset previous state
    iframe.onload = null;
    iframe.onerror = null;

    const showLoading = () => {
        if (config.showLoader && loadingDiv) {
            loadingDiv.style.display = 'block';
            loadingDiv.innerHTML = '<div class="jusdc-spinner"></div><p>Loading JUSDC Interface...</p>';
        }
        if (container) container.style.visibility = 'hidden';
    };

    const hideLoading = () => {
        if (loadingDiv) loadingDiv.style.display = 'none';
        if (container) container.style.visibility = 'visible';
    };

    const showError = (message) => {
        hideLoading();
        if (messageSpan) {
            messageSpan.textContent = message;
            messageSpan.style.color = '#ff4d4f';
            messageSpan.style.display = 'block';
        }
        
        // Add retry button if attempts remain
        if (config.currentAttempt < config.retryCount && messageSpan) {
            const retryBtn = document.createElement('button');
            retryBtn.textContent = 'Retry';
            retryBtn.className = 'jusdc-retry-btn';
            retryBtn.onclick = () => {
                messageSpan.innerHTML = '';
                loadWithRetry();
            };
            messageSpan.appendChild(document.createElement('br'));
            messageSpan.appendChild(retryBtn);
        }
    };

    const loadWithRetry = () => {
        config.currentAttempt++;
        
        // Add cache-busting parameter if enabled
        let finalUrl = url;
        if (config.cacheBust) {
            finalUrl += (url.includes('?') ? '&' : '?') + `_=${Date.now()}`;
        }

        showLoading();
        
        iframe.src = finalUrl;

        iframe.onload = () => {
            // Additional check for iframe content loaded properly
            try {
                // This check might need adjustment based on JUSDC's actual content
                if (iframe.contentDocument && iframe.contentDocument.body.children.length > 0) {
                    hideLoading();
                    if (messageSpan) {
                        messageSpan.textContent = '';
                        messageSpan.style.display = 'none';
                    }
                } else {
                    throw new Error('Empty iframe content');
                }
            } catch (e) {
                // Cross-origin check might fail, consider this successful
                hideLoading();
                if (messageSpan) {
                    messageSpan.textContent = '';
                    messageSpan.style.display = 'none';
                }
            }
        };

        iframe.onerror = () => {
            if (config.currentAttempt < config.retryCount) {
                console.warn(`JUSDC Loader: Attempt ${config.currentAttempt} failed, retrying...`);
                setTimeout(loadWithRetry, config.retryDelay);
            } else {
                console.error(`JUSDC Loader: All ${config.retryCount} attempts failed`);
                showError('Failed to load JUSDC interface. Please check your connection.');
            }
        };
    };

    loadWithRetry();
}

// JUSDC-specific initialization
document.addEventListener('DOMContentLoaded', () => {
    // Read Contract Interface
    loadJUSDCIframe('readcontractiframe', 'https://app.jusdc.io/read-contract', {
        retryCount: 5,
        retryDelay: 3000,
        cacheBust: true
    });

    // Write Contract Interface
    loadJUSDCIframe('writecontractiframe', 'https://app.jusdc.io/write-contract', {
        retryCount: 5,
        retryDelay: 3000,
        cacheBust: true
    });

    // Analytics Dashboard
    loadJUSDCIframe('analyticsiframe', 'https://analytics.jusdc.io/dashboard', {
        retryCount: 3,
        showLoader: false
    });

    // Add more JUSDC-specific iframes as needed
});

// Optional: Add CSS for JUSDC loader
const style = document.createElement('style');
style.textContent = `
    .jusdc-spinner {
        border: 4px solid rgba(0, 0, 0, 0.1);
        border-radius: 50%;
        border-top: 4px solid #2a52be;
        width: 30px;
        height: 30px;
        animation: spin 1s linear infinite;
        margin: 0 auto 10px;
    }
    @keyframes spin {
        0% { transform: rotate(0deg); }
        100% { transform: rotate(360deg); }
    }
    .jusdc-retry-btn {
        background-color: #2a52be;
        color: white;
        border: none;
        padding: 5px 15px;
        border-radius: 4px;
        cursor: pointer;
        margin-top: 10px;
    }
    .jusdc-retry-btn:hover {
        background-color: #1a3a8e;
    }
`;
document.head.appendChild(style);
