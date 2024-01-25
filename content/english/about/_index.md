---
title: "About Us"
meta_title: "About"
description: "Your trusted partner for innovative and professional DevOps services."
draft: false
---

<section class="about-us" data-aos="fade-up" data-aos-duration="1000"  style="width: 80%;">
  <h4>Welcome to Cloud Native Solutions, your trusted partner for innovative and professional DevOps services.</h4>

At Cloud Native Solutions, we take pride in our ability to stay at the forefront of industry trends and emerging technologies. We work closely with our clients to understand their unique challenges and goals, and leverage our expertise to design and implement customized solutions that deliver measurable results.

In 2021, we founded Cloud Native Solutions to help start-ups, small-to-medium-sized businesses, and enterprises streamline their software development processes and gain a competitive edge.

Marius Oprin, an experienced DevOps engineer, established the company after working closely with a medical automation platform CTO and team, who together, they envisioned a more reliable, affordable, and efficient way to manage their infrastructure.

Since then, our clientele has grown to include cloud computing, healthcare companies, marketing agencies, and email service providers, who all trust us to maintain high-performing platforms and ensure their cloud infrastructure is effective.

</section>

<!-- Swiper Slider Section -->
<section class="swiper-container" data-aos="fade-up" data-aos-duration="1000">
  <div class="swiper-wrapper">
    <div class="swiper-slide">
      <a href="https://cloudbase.it/" target="_blank">
        <img class="swiper-img grayscale cloud-base-img" src="https://i0.wp.com/cloudnativesolutions.ro/wp-content/uploads/2023/03/cloudbase.png?ssl=1" alt="Image 1">
      </a>
    </div>
    <div class="swiper-slide">
      <a href="https://www.revivalhealth.io/" target="_blank">
        <img class="swiper-img grayscale" src="https://i0.wp.com/cloudnativesolutions.ro/wp-content/uploads/2023/04/revivalhealth.png.jpeg?ssl=1" alt="Image 4">
      </a>
    </div>
    <div class="swiper-slide">
      <a href="https://www.opslogic.co/" target="_blank">
        <img class="swiper-img grayscale" src="https://i0.wp.com/cloudnativesolutions.ro/wp-content/uploads/2023/03/opslogic.png?ssl=1" alt="Image 2">
      </a>
    </div>
    <div class="swiper-slide">
      <a href="https://www.puresend.com/" target="_blank">
        <img class="swiper-img grayscale" src="https://i0.wp.com/cloudnativesolutions.ro/wp-content/uploads/2023/03/puresend.png?ssl=1" alt="Image 3">
      </a>
    </div>
    <div class="swiper-slide">
      <a href="https://www.vitalinteraction.com/" target="_blank">
        <img class="swiper-img grayscale" src="https://i0.wp.com/cloudnativesolutions.ro/wp-content/uploads/2023/04/Untitled-design-11.png?ssl=1" alt="Image 5">
      </a>
    </div>
  </div>
</section>

<script src="https://unpkg.com/swiper/swiper-bundle.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/aos@2.3.4/dist/aos.js"></script>
<script>
  document.addEventListener('DOMContentLoaded', function() {
    AOS.init();
    
    var swiper = new Swiper('.swiper-container', {
  slidesPerView: 5,
  spaceBetween: 50,
  loop: true,
  autoplay: {
    delay: 2500,
  },
  breakpoints: {
     280: {
      slidesPerView: 1, 
      spaceBetween: 10,
    },
    320: {
      slidesPerView: 1, 
      spaceBetween: 15,
    },
    480: {
      slidesPerView: 2, 
      spaceBetween: 30,
    },
    768: {
      slidesPerView: 3, 
      spaceBetween: 50,
    },
    1024: {
      slidesPerView: 5, 
      spaceBetween: 30,
    },
  }
});


    // Grayscale to Color on Hover
    var swiperImgs = document.querySelectorAll('.swiper-img');
    swiperImgs.forEach(function(img) {
      img.addEventListener('mouseenter', function() {
        img.classList.remove('grayscale');
      });

      img.addEventListener('mouseleave', function() {
        img.classList.add('grayscale');
      });

      img.addEventListener('click', function() {
        window.open(img.parentElement.href, '_blank');
      });
    });
  });
</script>

<style>
  /* Swiper Styles */
  .swiper-container {
    width: 100%;
    overflow: hidden;
    padding: 5px 15px;
    /* background: #c3a9b97a; */
    border-radius: 30px;
  }

  .swiper-wrapper{
    display: flex;
    width: 100%;
    align-items: center;
  }

  .swiper-img {
    width: 80%;
    object-fit: cover;
    transition: filter 0.5s ease;
  }

  .cloud-base-img{
    background: white;
  }

  .grayscale {
    filter: grayscale(100%);
  }

    :root {
      --tw-gradient-from: rgba(249, 249, 249, 1);
      --tw-gradient-from-position: 0.53%;
      --tw-gradient-to: rgba(249, 249, 249, 0);
      --tw-gradient-to-position: 83.28%;
      --tw-gradient-stops: var(--tw-gradient-from) var(--tw-gradient-from-position), var(--tw-gradient-to) var(--tw-gradient-to-position);
    }

  .our-expertise{
 background-image: linear-gradient(to bottom, var(--tw-gradient-stops));
 margin-top: 1rem;
  }

</style>

<!-- Content Section -->
<section class="our-expertise" style="width: 100%;">

  <h2>Our Expertise</h2>
  <p>
    Our team boasts a wealth of experience in Kubernetes management, cloud management, CI/CD pipelines, infrastructure design, and container solutions. We are well-versed in all major cloud providers, such as AWS, GCP, and Azure.
  </p>
  <p style=" margin-bottom: 2rem;">
    We are committed to providing our clients with the best possible cloud solutions and ensuring they sleep well at night, knowing their platforms are in good hands.
  </p>

<a href="/services/" class="btn btn-primary my-btn-style"> Learn more about our services here</a>

</section>

<section style="width: 80%;">

 <h2 style="margin-top: 5rem;">HOW WE WORK</h2>

  <h3>Fully Managed DevOps Services</h3>
  <p>
    We will serve as your internal DevOps team, responsible for designing, managing, and preserving your infrastructure, regardless of whether you have a pre-existing platform or are in the early stages of product development.
  </p>
  <p>
    We will provide our own recommended tools, procedural guidelines, and proposed work methods, while tailoring our strategy to accommodate your unique requirements.
  </p>

  <h3>DevOps Team Augmentation</h3>
  <p>
    In case you lack the time or resources to undergo the process of recruiting new personnel, we can offer supplementary assistance to your current in-house DevOps team. We will enhance the existing team with specialized skills and knowledge, along with extra resources, to aid in managing infrastructure and streamlining processes. We provide effective support to help you manage your backlog, freeing up your time and energy to focus on new opportunities and growth for your business.
  </p>
</section>
