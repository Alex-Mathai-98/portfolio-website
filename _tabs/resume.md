---
layout: page-no-sidebar
icon: fas fa-archive
order: 4
---

<style>
.dynamic-title {
  padding: 5% 10% 0 10% !important;
  max-width: 1200px !important;
  margin: 0 auto !important;
}
</style>

<div style="padding: 0 10%; max-width: 1200px; margin: 0 auto;">

<style>
/* Desktop PDF viewer */
.pdf-viewer-desktop {
  display: block;
}

.pdf-viewer-mobile {
  display: none;
}

/* Mobile-specific styles */
@media (max-width: 768px) {
  .pdf-viewer-desktop {
    display: none;
  }
  
  .pdf-viewer-mobile {
    display: block;
    text-align: center;
    padding: 40px 20px;
  }
  
  .mobile-pdf-button {
    display: inline-block;
    background-color: #007bff;
    color: white;
    padding: 15px 30px;
    text-decoration: none;
    border-radius: 8px;
    font-size: 18px;
    font-weight: bold;
    margin: 20px 0;
    box-shadow: 0 4px 8px rgba(0,0,0,0.2);
    transition: background-color 0.3s;
  }
  
  .mobile-pdf-button:hover {
    background-color: #0056b3;
    color: white;
    text-decoration: none;
  }
}
</style>

<!-- Desktop PDF Viewer -->
<div class="pdf-viewer-desktop">
  <div style="width: 100%; height: 80vh;">
    <object data="{{ '/assets/pdf/Alex_Mathai_CV2.pdf' | relative_url }}" type="application/pdf" width="100%" height="100%">
      <iframe src="https://mozilla.github.io/pdf.js/web/viewer.html?file={{ '/assets/pdf/Alex_Mathai_CV2.pdf' | absolute_url }}" width="100%" height="100%">
        <p>Your browser doesn't support PDF viewing. <a href="{{ '/assets/pdf/Alex_Mathai_CV2.pdf' | relative_url }}" target="_blank">Download the PDF</a></p>
      </iframe>
    </object>
  </div>
</div>

<!-- Mobile PDF Viewer -->
<div class="pdf-viewer-mobile">
  <h2>Resume / CV</h2>
  <p>View my complete resume by opening the PDF below:</p>
  <a href="{{ '/assets/pdf/Alex_Mathai_CV2.pdf' | relative_url }}" target="_blank" class="mobile-pdf-button">
    📄 View Resume PDF
  </a>
  <p><small>The PDF will open in your browser's built-in viewer</small></p>
</div>

</div>