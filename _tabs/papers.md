---
layout: page-no-sidebar
icon: fas fa-info-circle
order: 0
title: 
---

<style>
.dynamic-title {
  padding: 5% 10% 0 10% !important;
  max-width: 1200px !important;
  margin: 0 auto !important;
}

/* AIDEV-NOTE: pill badge styles for paper/project links */
.badge-paper, .badge-project {
  display: inline-block;
  padding: 6px 14px;
  border-radius: 20px;
  font-size: 1em;
  vertical-align: middle;
  font-weight: 600;
  text-decoration: none;
  letter-spacing: 0.03em;
  transition: opacity 0.2s ease;
}
.badge-paper:hover, .badge-project:hover {
  opacity: 0.85;
  text-decoration: none;
}
.badge-paper {
  background-color: #0073aa;
  color: #fff !important;
}
.badge-project {
  background-color: #28a745;
  color: #fff !important;
}
/* AIDEV-NOTE: conditional sizing for KBench Columbia logo */
.kbench-logo {
  max-width: 200px;
}

/* AIDEV-NOTE: ornamental flourish divider between paper sections */
.section-divider {
  display: flex;
  align-items: center;
  text-align: center;
  margin: 10px 0 40px 0;
  color: #b0b0b0;
}
.section-divider::before,
.section-divider::after {
  content: '';
  flex: 1;
  height: 1px;
  background: linear-gradient(to var(--dir), transparent, #c0c0c0);
}
.section-divider::before {
  --dir: right;
  margin-right: 16px;
}
.section-divider::after {
  --dir: left;
  margin-left: 16px;
}
.section-divider span {
  font-size: 1.2em;
  letter-spacing: 8px;
  color: #b0b0b0;
}

.resources-label {
  font-size: 1em;
  font-weight: 600;
  color: #5ba4cf;
  margin-right: 8px;
  letter-spacing: 0.03em;
  vertical-align: middle;
}

/* AIDEV-NOTE: mobile responsive rules mirrored from index.md home page */
@media (max-width: 768px) {
  /* Stack side-by-side paper sections vertically */
  div[style*="display: flex"][style*="align-items"] {
    flex-direction: column !important;
  }
  /* Reduce outer padding */
  div[style*="padding: 0 10%"] {
    padding: 0 5% !important;
  }
  /* Make stacked children full-width */
  div[style*="display: flex"][style*="align-items"] > div {
    width: 100% !important;
  }
  /* Center only image containers */
  div[style*="display: flex"][style*="align-items"] > div:has(img) {
    text-align: center !important;
  }
  /* Always show image above text when stacked */
  div[style*="display: flex"][style*="align-items"] > div:has(img) {
    order: -1 !important;
  }
  /* Cap and center images — use max-width instead of width so logos don't blow up */
  img {
    max-width: 200px !important;
    height: auto !important;
    display: block !important;
    margin: 0 auto 20px auto !important;
  }
  /* Constrain stacked sections to 80% width, centered */
  div[style*="display: flex"][style*="align-items"] > div {
    max-width: 80% !important;
    margin: 0 auto !important;
  }
  /* Shrink paper title headings */
  h3[style*="font-size: 1.35em"] {
    font-size: 1.1em !important;
  }
  /* Titles and badges are already center-aligned, no override needed */
  .kbench-logo {
    max-width: 180px !important;
  }
}
</style>


<div style="padding: 0 10%; max-width: 1200px; margin: 0 auto;">


<div style="text-align: center; margin: 40px 0;">
  <!-- <h2 style="font-size: 2.5em; font-weight: bold; margin-bottom: 40px;"> Work Experience | Projects </h2> -->
  <div style="height: 3px; background-color: #333; margin: 0 auto 60px auto; width: 200px;"></div>
</div>

<h3 style="font-size: 1.35em; font-weight: 600; margin-top: 0; margin-bottom: 6px; text-align: center;">KBench & KGym - A Benchmark And Platform To Test LLMs On Linux Kernel Bug Resolution</h3>
<div style="margin-bottom: 20px; text-align: center;"><span class="resources-label">Resources</span><a href="https://arxiv.org/abs/2407.02680" class="badge-paper">Paper</a> <a href="https://github.com/Alex-Mathai-98/kGym-Kernel-Gym" class="badge-project">Project</a></div>
<div style="display: flex; align-items: flex-start; gap: 30px; margin-bottom: 60px;">
  <div style="flex: 1;">
    <a href="https://arxiv.org/abs/2407.02680"><img src="/assets/img/columbia-logo.jpg" alt="Columbia University" class="kbench-logo" style="width: 100%; border-radius: 8px; border: 4px solid transparent; box-shadow: 0 0 8px 4px rgba(0, 115, 170, 0.4), 0 0 20px 8px rgba(0, 115, 170, 0.1);"></a>
  </div>
  <div style="flex: 2.5;">
    <div style="text-align: justify; font-size: 1.2em; line-height: 1.5;">
      I worked with Chenxi Huang to establish the first benchmark and platform that tests LLMs on bug resolution in the Linux kernel. Through our experiments, we showed that LLMs have a large scope for improvement when resolving bugs in low-level and complicated software like kernel code.
    </div>
  </div>
</div>

<div class="section-divider"><span>&#10022;</span></div>

<h3 style="font-size: 1.35em; font-weight: 600; margin-top: 0; margin-bottom: 6px; text-align: center;">COMEX - Generating Customized Source Code Representations</h3>
<div style="margin-bottom: 20px; text-align: center;"><span class="resources-label">Resources</span><a href="https://github.com/IBM/tree-sitter-codeviews" class="badge-project">Project</a></div>
<div style="display: flex; align-items: flex-start; gap: 30px; margin-bottom: 60px;">
  <div style="flex: 2.5;">
    <div style="text-align: justify; font-size: 1.2em; line-height: 1.5;">
      I worked with Debeshee Das, Noble Saji Mathews, and Srikanth Tamilselvam (Manager, IBM Research) on creating tools that generate customized source code representations for any generic code snippet (i.e. complete, incomplete, or uncompilable code). This capability is very useful when we wish to use static analysis on incomplete code being fed to an LLM.
    </div>
  </div>
    <div style="flex: 1;">
    <a href="https://github.com/IBM/tree-sitter-codeviews"><img src="/assets/wordpress/webiste_image.png" alt="COMEX Tool" style="width: 100%; max-width: 200px; border-radius: 15px;"></a>
  </div>
</div>

<div class="section-divider"><span>&#10022;</span></div>

<h3 style="font-size: 1.35em; font-weight: 600; margin-top: 0; margin-bottom: 6px; text-align: center;">Graph Neural Networks For The Recommendation Of Candidate Microservices</h3>
<div style="margin-bottom: 20px; text-align: center;"><span class="resources-label">Resources</span><a href="https://www.ijcai.org/proceedings/2022/0542.pdf" class="badge-paper">Paper</a></div>
<div style="display: flex; align-items: flex-start; gap: 30px; margin-bottom: 60px;">
  <div style="flex: 1;">
    <a href="https://www.ijcai.org/proceedings/2022/0542.pdf"><img src="/assets/wordpress/webiste_image.png" alt="Graph Neural Networks" style="width: 100%; max-width: 200px; border-radius: 15px;"></a>
  </div>
  <div style="flex: 2.5;">
    <div style="text-align: justify; font-size: 1.2em; line-height: 1.5;">
      I worked with Srikanth Tamilselvam (Manager, IBM Research) on the <strong>Candidate Microservice Advisor</strong> project. In this research thread, we experimented with different techniques to represent application software as graphs, which we then partition into smaller sized groups using clustering mechanisms. To this end, I helped in translating this decomposition task as a constrained clustering problem over an embedding space learnt by a heterogeneous graph neural network.
    </div>
  </div>
</div>

<div class="section-divider"><span>&#10022;</span></div>

<h3 style="font-size: 1.35em; font-weight: 600; margin-top: 0; margin-bottom: 6px; text-align: center;">Knowledge Graph Modelling For Mainframe Application Modernization</h3>
<div style="margin-bottom: 20px; text-align: center;"><span class="resources-label">Resources</span><a href="https://dl.acm.org/doi/abs/10.1145/3493700.3493735" class="badge-paper">Paper</a></div>
<div style="display: flex; align-items: flex-start; gap: 30px; margin-bottom: 60px;">
  <div style="flex: 2.5;">
    <div style="text-align: justify; font-size: 1.2em; line-height: 1.5;">
      I worked with Amith Singhee (Director, IBM Research) on creating research tools that simplify how we modernize legacy applications. I helped model legacy mainframe codebases as a very fine-grained knowledge graph (KG). Using this KG, we developed methods that allow application architects to make data driven decisions. Such informed decisions allow for a smooth incremental modernization journey of legacy codebases.
    </div>
  </div>
  <div style="flex: 1;">
    <a href="https://dl.acm.org/doi/abs/10.1145/3493700.3493735"><img src="/assets/wordpress/webiste_image.png" alt="Knowledge Graph" style="width: 100%; max-width: 200px; border-radius: 15px;"></a>
  </div>
</div>

<div class="section-divider"><span>&#10022;</span></div>

<h3 style="font-size: 1.35em; font-weight: 600; margin-top: 0; margin-bottom: 6px; text-align: center;">Network Traffic Classification And Estimating User Experience</h3>
<div style="margin-bottom: 20px; text-align: center;"><span class="resources-label">Resources</span><a href="https://ieeexplore.ieee.org/abstract/document/9521288" class="badge-paper">Paper</a></div>
<div style="display: flex; align-items: flex-start; gap: 30px; margin-bottom: 60px;">
  <div style="flex: 1.5;">
    <a href="https://ieeexplore.ieee.org/abstract/document/9521288"><img src="/assets/wordpress/UNSW.png" alt="UNSW Sydney" style="width: 100%; max-width: 800px; border-radius: 15px;"></a>
  </div>
  <div style="flex: 2.5;">
    <div style="text-align: justify; font-size: 1.2em; line-height: 1.5;">
      I worked on the classification of encrypted network traffic and on estimating user experience in internet applications. The network domain proves to be much more challenging than language and vision because of infinitely many error-inducing factors like network congestion, different network architectures, and various bandwidth capacities. I was able to research and extract robust and reliable features that are immune to varying conditions, and that provide a clear signal for fast and accurate classification.
    </div>
  </div>
</div>

<div class="section-divider"><span>&#10022;</span></div>

<h3 style="font-size: 1.35em; font-weight: 600; margin-top: 0; margin-bottom: 6px; text-align: center;">Adversarial Black-Box Attacks On Text Classifiers Using Genetic Algorithms Guided By Deep Networks</h3>
<div style="margin-bottom: 20px; text-align: center;"><span class="resources-label">Resources</span><a href="https://arxiv.org/abs/2011.03901" class="badge-paper">Paper</a></div>
<div style="display: flex; align-items: flex-start; gap: 30px; margin-bottom: 60px;">
  <div style="flex: 2.5;">
    <div style="text-align: justify; font-size: 1.2em; line-height: 1.5;">
      My work predominantly focused on studying the robustness of popular text classification models against adversarial attacks. I successfully created adversarial examples using genetic algorithms guided by deep neural networks for many state-of-the-art text classifiers including BERT, RoBERTa, and DistilBERT. The research contributed to understanding vulnerabilities in natural language processing models and improving their defensive capabilities.
    </div>
  </div>
  <div style="flex: 1.5;">
    <a href="#"><img src="/assets/wordpress/adversarialtext16.png" alt="Adversarial Attacks" style="width: 100%; max-width: 800px; border-radius: 15px;"></a>
  </div>
</div>

</div>
