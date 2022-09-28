<script>
  import { getContext, createEventDispatcher, onMount } from "svelte";
  import { flip } from "svelte/animate";
  import AppData from "../stores/AppData.js";
  import HeroData from "../stores/HeroData.js";
  import HeroDetail from "../modals/HeroDetail.svelte";
  import ModalCloseButton from "../modals/ModalCloseButton.svelte";
  import AscendBox from "../shared/AscendBox.svelte";
  import SIFurnEngBox from "../shared/SIFurnEngBox.svelte";
  import StarsInput from "../shared/StarsInput.svelte";
  import { validateJWT } from "../rest/RESTFunctions.svelte";

  const { open } = getContext("simple-modal");
  const dispatch = createEventDispatcher();

  export let isMobile = false;

  let recommendations = buildRecs();
  const sections = ["Sig. Item", "Furniture", "Eng."];

  $: modalHeight = isMobile ? "75vh" : "80vh";

  onMount(async () => {
    $AppData.activeView = "benchmarkmax";
    dispatch("routeEvent", { action: "saveData" });
  });

  // compare compHeroes with MH.List and create recommendation objects where values differ
  function buildRecs() {
    let buffer = [];
    let hero;
    for (let i = 0; i < $HeroData.length; i++) {
      hero = $HeroData[i];
      if ($AppData.MH.List[hero.id].claimed) {
        if (
          $AppData.MH.List[hero.id].si < hero.si_benchmark_max &&
          hero.si_benchmark_max > 0
        ) {
          buffer.push({
            id: hero.id,
            type: "si",
            value: { si: hero.si_benchmark_max },
          });
        }
        if ($AppData.MH.List[hero.id].furn < hero.furn_benchmark_max) {
          buffer.push({
            id: hero.id,
            type: "furn",
            value: { furn: hero.furn_benchmark_max },
          });
        }
        if (
          $AppData.MH.List[hero.id].engraving < hero.engraving_benchmark_max
        ) {
          buffer.push({
            id: hero.id,
            type: "engraving",
            value: {
              engraving: hero.engraving_benchmark_max,
              stars: $AppData.MH.List[hero.id].stars,
            },
          });
        }
      }
    }
    return buffer;
  }

  async function postUpdate() {
    $AppData.MH.lastUpdate = new Date();
    const valid = await validateJWT($AppData.user.jwt);
    if (valid) dispatch("routeEvent", { action: "syncMyHeroes" });
  }

  function sortByCore(a, b) {
    if (a.core && !b.core) {
      return -1;
    } else if (!a.core && b.core) {
      return 1;
    } else {
      return 0;
    }
  }

  function handlePortraitClick(heroID) {
    const bgColor = window
      .getComputedStyle(document.documentElement)
      .getPropertyValue("--appBGColor");
    open(
      HeroDetail,
      { heroID: heroID },
      {
        closeButton: ModalCloseButton,
        styleWindow: { background: bgColor },
        styleContent: {
          background: bgColor,
          padding: 0,
          borderRadius: "10px",
          maxHeight: modalHeight,
        },
      }
    );
  }

  async function handleClaimClick(heroID, value, type) {
    $AppData.MH.List[heroID].claimed = true;
    switch (type) {
      case "si":
        // allow claiming of si levels before mythic but set to mythic automatically
        if ($AppData.MH.List[heroID].ascendLv < 4)
          $AppData.MH.List[heroID].ascendLv = 4;
        $AppData.MH.List[heroID].si = value.si;
        break;
      case "furn":
        // allow claiming of furniture levels before ascended but set to ascended automatically
        if ($AppData.MH.List[heroID].ascendLv < 6)
          $AppData.MH.List[heroID].ascendLv = 6;
        $AppData.MH.List[heroID].furn = value.furn;
        break;
      case "engraving":
        // allow claiming of engraving levels before ascended but set to ascended automatically
        if ($AppData.MH.List[heroID].ascendLv < 6)
          $AppData.MH.List[heroID].ascendLv = 6;
        // allow claiming of engraving levels with 0* but set stars to recommended stars
        if ($AppData.MH.List[heroID].stars < value.stars)
          $AppData.MH.List[heroID].stars = value.stars;
        $AppData.MH.List[heroID].engraving = value.engraving;
        break;
      default:
        throw new Error(
          `Invalid type received ${type} should be 'asc', 'si', or 'furn'.`
        );
    }
    recommendations = buildRecs();
    await postUpdate();
    dispatch("routeEvent", { action: "saveData" });
  }
</script>

<div class="recContainer">
  <div class="sectionPickerSection">
    <ul class="sectionPicker">
      {#each sections as section, i}
        <li>
          <button
            type="button"
            class="sectionButton"
            class:active={$AppData.REC.openSection === i}
            on:click={() => {
              $AppData.REC.openSection = i;
              dispatch("routeEvent", { action: "saveData" });
            }}
          >
            <span>{section}</span>
          </button>
        </li>
      {/each}
    </ul>
  </div>
  {#if $AppData.REC.openSection === 0}
    <section class="recSection siSection">
      <div class="recArea">
        {#if recommendations.filter((e) => e.type === "si").length > 0}
          {#each recommendations
            .filter((e) => e.type === "si")
            .sort(sortByCore) as rec (rec.id + "_si")}
            <div class="recCard" animate:flip={{ duration: 200 }}>
              <div class="claimButtonArea">
                <button
                  type="button"
                  class="claimButton"
                  on:click={handleClaimClick(rec.id, rec.value, "si")}
                  >&#10004;</button
                >
              </div>
              <div class="portraitContainer">
                <button
                  type="button"
                  class="portraitButton"
                  on:click={() => handlePortraitClick(rec.id)}
                >
                  <img
                    class="portrait"
                    src={$HeroData.find((e) => e.id === rec.id).portrait}
                    alt={$HeroData.find((e) => e.id === rec.id).name}
                  />
                  <span class="coreMark" class:visible={rec.core} />
                </button>
              </div>
              <h4>{$HeroData.find((e) => e.id === rec.id).name}</h4>
              <div class="recText">
                <SIFurnEngBox
                  type="si"
                  num={rec.value.si}
                  fullName={true}
                  maxWidth="100px"
                />
              </div>
            </div>
          {/each}
        {:else}
          <div class="noRec">
            <span>No Signature Item Recommendations</span>
          </div>
        {/if}
      </div>
    </section>
  {:else if $AppData.REC.openSection === 1}
    <section class="recSection furnSection">
      <div class="recArea">
        {#if recommendations.filter((e) => e.type === "furn").length > 0}
          {#each recommendations
            .filter((e) => e.type === "furn")
            .sort(sortByCore) as rec (rec.id + "_furn")}
            <div class="recCard" animate:flip={{ duration: 200 }}>
              <div class="claimButtonArea">
                <button
                  type="button"
                  class="claimButton"
                  on:click={handleClaimClick(rec.id, rec.value, "furn")}
                  >&#10004;</button
                >
              </div>
              <div class="portraitContainer">
                <button
                  type="button"
                  class="portraitButton"
                  on:click={() => handlePortraitClick(rec.id)}
                >
                  <img
                    class="portrait"
                    src={$HeroData.find((e) => e.id === rec.id).portrait}
                    alt={$HeroData.find((e) => e.id === rec.id).name}
                  />
                  <span class="coreMark" class:visible={rec.core} />
                </button>
              </div>
              <h4>{$HeroData.find((e) => e.id === rec.id).name}</h4>
              <div class="recText">
                <SIFurnEngBox type="furn" num={rec.value.furn} />
              </div>
            </div>
          {/each}
        {:else}
          <div class="noRec"><span>No Furniture Recommendations</span></div>
        {/if}
      </div>
    </section>
  {:else if $AppData.REC.openSection === 2}
    <section class="recSection engSection">
      <div class="recArea">
        {#if recommendations.filter((e) => e.type === "engraving").length > 0}
          {#each recommendations
            .filter((e) => e.type === "engraving")
            .sort(sortByCore) as rec (rec.id + "_eng")}
            <div class="recCard" animate:flip={{ duration: 200 }}>
              <div class="claimButtonArea">
                <button
                  type="button"
                  class="claimButton"
                  on:click={handleClaimClick(rec.id, rec.value, "engraving")}
                  >&#10004;</button
                >
              </div>
              <div class="portraitContainer">
                <button
                  type="button"
                  class="portraitButton"
                  on:click={() => handlePortraitClick(rec.id)}
                >
                  <img
                    class="portrait"
                    src={$HeroData.find((e) => e.id === rec.id).portrait}
                    alt={$HeroData.find((e) => e.id === rec.id).name}
                  />
                  <span class="coreMark" class:visible={rec.core} />
                </button>
              </div>
              <h4>{$HeroData.find((e) => e.id === rec.id).name}</h4>
              <div class="starsInputContainer">
                <StarsInput
                  value={rec.value.stars}
                  enabled={true}
                  engraving={rec.value.engraving}
                  displayOnly={true}
                />
              </div>
              <div class="recText">
                <SIFurnEngBox type="engraving" num={rec.value.engraving} />
              </div>
            </div>
          {/each}
        {:else}
          <div class="noRec"><span>No Engraving Recommendations</span></div>
        {/if}
      </div>
    </section>
  {/if}
</div>

<style lang="scss">
  .recContainer {
    height: 100%;
    height: calc(
      var(--vh, 1vh) * 100 - var(--headerHeight)
    ); /* gymnastics to set height for mobile browsers */
    padding: 10px;
    overflow-y: auto;
    width: 100%;
  }
  .sectionPickerSection {
    margin: 10px;
    width: 95%;
  }
  .sectionPicker {
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    justify-content: center;
    list-style-type: none;
    margin: 0;
    padding: 0;
    .sectionButton {
      align-items: center;
      background: var(--appBGColor);
      border: none;
      border-radius: 10px;
      box-shadow: var(--neu-med-i-BGColor-shadow);
      color: var(--appColorPrimary);
      cursor: pointer;
      display: flex;
      font-size: 1.2rem;
      justify-content: center;
      margin: 5px 8px;
      outline: none;
      padding: 5px;
      span {
        opacity: 0.5;
      }
      &.active {
        background: var(--neu-convex-BGColor-bg);
        span {
          opacity: 1;
        }
      }
    }
  }
  .recSection {
    padding: 10px;
    width: 100%;
  }
  .recArea {
    display: grid;
    grid-gap: 20px 20px;
    grid-template-columns: repeat(auto-fit, minmax(250px, 270px));
    justify-content: space-evenly;
    margin-bottom: 10px;
  }
  .noRec {
    align-items: center;
    display: flex;
    height: 100%;
    justify-content: center;
    width: 100%;
    user-select: none;
    span {
      color: rgba(100, 100, 100, 0.3);
      font-size: 2.2rem;
      font-weight: bold;
      text-align: center;
      text-transform: uppercase;
    }
  }
  .recCard {
    background-color: var(--appBGColor);
    border: none;
    border-radius: 10px;
    box-shadow: var(--neu-med-i-BGColor-shadow);
    padding: 10px;
    position: relative;
    h4 {
      font-size: 1.2rem;
      margin: 0;
      margin-bottom: 3px;
      text-align: center;
      width: 100%;
    }
    .starsInputContainer {
      display: flex;
      justify-content: center;
    }
  }
  .claimButtonArea {
    position: absolute;
    right: 5px;
    top: 5px;
    z-index: 1;
    .claimButton {
      background-color: transparent;
      border: none;
      border-radius: 50%;
      box-shadow: var(--neu-sm-i-BGColor-pressed-shadow);
      color: var(--appColorPrimary);
      cursor: pointer;
      font-size: 0.9rem;
      height: 25px;
      outline: none;
      padding: 0px;
      width: 25px;
    }
  }
  .portraitContainer {
    align-items: center;
    display: flex;
    justify-content: center;
    position: relative;
    width: 100%;
    .portraitButton {
      background: transparent;
      border: none;
      cursor: pointer;
      outline: none;
      .portrait {
        border-radius: 50%;
        max-width: 100px;
      }
    }
    .coreMark {
      background-color: var(--legendColor);
      border: 4px solid var(--appBGColor);
      border-radius: 50%;
      bottom: 2px;
      display: none;
      height: 28px;
      position: absolute;
      right: 75px;
      visibility: hidden;
      width: 28px;
    }
    .coreMark.visible {
      display: inline-block;
      pointer-events: none;
      visibility: visible;
    }
  }
  .recText {
    display: flex;
    justify-content: center;
    margin: 10px 0px;
    width: 100%;
  }
  @media only screen and (min-width: 767px) {
    .recContainer {
      height: 100vh;
    }
    .sectionPicker {
      justify-content: flex-start;
      li {
        &:first-child {
          .sectionButton {
            margin-left: 0px;
          }
        }
      }
      .sectionButton {
        &:hover {
          background: var(--neu-convex-BGColor-wide-bg);
        }
      }
    }
    .recSection {
      border-radius: 0px 10px 10px 10px;
    }
    .noRec {
      span {
        font-size: 6rem;
      }
    }
    .claimButton {
      &:hover {
        background: var(--neu-convex-BGColor-bg);
      }
    }
  }
</style>
