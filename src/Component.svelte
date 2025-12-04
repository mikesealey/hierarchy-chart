<script>
  import { getContext, onDestroy } from "svelte";
  import ApexTree from "apextree";

  const { styleable } = getContext("sdk");
  const component = getContext("component");
  let container;

  // Settings from schema
  export let records;
  export let dataProvider;
  export let text1Column;
  export let text2Column;
  export let text3Column;
  export let imageColumn;
  export let reportsToColumn;
  export let topOfTree;
  export let cardWidth;
  export let cardHeight;
  export let siblingSpacing;
  export let childrenSpacing;

  console.log("Data Provider", dataProvider)
  console.log("image column", imageColumn)
  console.log("reportsToColumn", reportsToColumn)

  // Node template used by ApexTree
  function nodeTemplate(content) {
    const text1 = content.text1 || "";
    const text2 = content.text2 || "";
    const text3 = content.text3 || "";
    const img = content.imageURL || "";

    return `
      <div class="card">

        ${img
          ? `<img class="card-img" src="${img}"/>`
          : ``}

        <div class="card-text-1">${text1}</div>
        <div class="card-text-2">${text2}</div>
        <div class="card-text-3">${text3}</div>

      </div>
    `;
  }

  // Make options reactive so changes are picked up on remount
  let options;
  $: options = {
    height: 800,
    width: 1000,
    nodeWidth: cardWidth,
    nodeHeight: cardHeight,
    childrenSpacing,
    siblingSpacing,
    direction: "top",
    contentKey: "data",
    nodeTemplate,
    canvasStyle: "border: 1px solid #ccc;",
    borderRadius: "15px"

  };

  function getCell(row, key) {
    if (!key) return undefined;
    return row?.[key];
  }

  function getFirstRelId(value) {
    // relationship values are typically arrays of related rows
    if (!value) return undefined;
    if (typeof value === "string") return value; // sometimes stored as id
    const rel = Array.isArray(value) ? value[0] : value;
    if (rel && typeof rel === "object") {
      return rel._id || rel.id;
    }
    return undefined;
  }

  function normaliseImage(value) {
    // Accept string URL, attachment object with url, or array
    if (!value) return null;
    if (typeof value === "string") return value;
    if (Array.isArray(value)) return normaliseImage(value[0]);
    if (typeof value === "object" && value.url) return value.url;
    return null;
  }

  function looksLikeRelationship(val) {
    if (!val) return false;
    if (Array.isArray(val)) {
      return val.length > 0 && typeof val[0] === "object" && ("_id" in val[0] || "id" in val[0]);
    }
    return typeof val === "object" && ("_id" in val || "id" in val);
  }

  function detectSelfRelationKey(rows, excludeKey) {
    const preferred = ["users_table", "user", "users", "owner", "person"];
    for (const key of preferred) {
      if (key === excludeKey) continue;
      if (rows.some(r => looksLikeRelationship(r?.[key]))) {
        return key;
      }
    }
    // Fallback: find any relationship-like key present on many rows
    const counts = {};
    rows.forEach(r => {
      Object.keys(r || {}).forEach(k => {
        if (k === excludeKey) return;
        if (looksLikeRelationship(r[k])) {
          counts[k] = (counts[k] || 0) + 1;
        }
      });
    });
    let bestKey = null;
    let bestCount = 0;
    for (const [k, count] of Object.entries(counts)) {
      if (count > bestCount) {
        bestKey = k;
        bestCount = count;
      }
    }
    return bestCount > 0 ? bestKey : null;
  }

  function buildOrgTree(rows) {
    if (!Array.isArray(rows) || rows.length === 0) {
      return null;
    }

    // Index rows and detect self relationship key for cross-table mapping
    const rowsById = {};
    rows.forEach(r => (rowsById[r?._id] = r));
    const externalIdToRowId = {};
    const selfRelKey = detectSelfRelationKey(rows, reportsToColumn);

    // Prepare node content but don't link yet
    const nodeDataById = {};
    rows.forEach(row => {
      const text1 = getCell(row, text1Column) ?? "";
      const text2 = getCell(row, text2Column) ?? "";
      const text3 = getCell(row, text3Column) ?? "";
      const rawImage = getCell(row, imageColumn);
      const imageURL = typeof rawImage === "string" ? rawImage : normaliseImage(rawImage);

      nodeDataById[row._id] = { id: row._id, text1, text2, text3, imageURL };

      if (selfRelKey) {
        const selfRelVal = getCell(row, selfRelKey);
        const externalId = getFirstRelId(selfRelVal);
        if (externalId) {
          externalIdToRowId[externalId] = row._id;
        }
      }
    });

    // Compute parent relationships for each row
    const parentOf = {};
    rows.forEach(row => {
      const relVal = getCell(row, reportsToColumn);
      const parentRefId = typeof relVal === "string" ? relVal : getFirstRelId(relVal);
      let parentRowId = parentRefId;
      // Resolve parent: prefer direct same-table id, else map via externalIdToRowId
      if (parentRowId && !rowsById[parentRowId] && externalIdToRowId[parentRowId]) {
        parentRowId = externalIdToRowId[parentRowId];
      }
      if (parentRowId && parentRowId !== row._id) {
        parentOf[row._id] = parentRowId;
      }
    });

    // Invert to children map
    const childrenById = {};
    Object.entries(parentOf).forEach(([childId, parentId]) => {
      if (!childrenById[parentId]) childrenById[parentId] = [];
      childrenById[parentId].push(childId);
    });

    // Determine root id
    let rootId = null;
    if (topOfTree && rowsById[topOfTree]) {
      rootId = topOfTree;
    } else {
      // Find any id that is not a child
      const childIds = new Set(Object.keys(parentOf));
      rootId = rows.find(r => !childIds.has(r._id))?._id || rows[0]._id;
    }

    // If topOfTree is set, restrict to its descendants only
    const allowed = new Set();
    const stack = [];
    if (rootId) {
      allowed.add(rootId);
      stack.push(rootId);
    }
    while (stack.length) {
      const current = stack.pop();
      const kids = childrenById[current] || [];
      for (const cid of kids) {
        if (!allowed.has(cid)) {
          allowed.add(cid);
          stack.push(cid);
        }
      }
    }

    // Build nodes only for allowed set and link
    const nodes = {};
    allowed.forEach(id => {
      const data = nodeDataById[id] || { id };
      nodes[id] = { id, data, options: {}, children: [] };
    });
    allowed.forEach(pid => {
      const kids = childrenById[pid] || [];
      kids.forEach(cid => {
        if (allowed.has(cid)) {
          nodes[pid].children.push(nodes[cid]);
        }
      });
    });

    return nodes[rootId] || null;
  }

  let tree;

  // Separate dependencies for data and layout
  let dataDeps;
  $: dataDeps = [
    records,
    dataProvider,
    text1Column,
    text2Column,
    text3Column,
    imageColumn,
    reportsToColumn,
    topOfTree
  ];

  // Key used to force remount when layout-affecting props change
  let layoutKey;
  $: layoutKey = JSON.stringify([
    cardWidth,
    cardHeight,
    siblingSpacing,
    childrenSpacing
  ]);

  // Initialize ApexTree whenever the container is (re)created
  $: if (container) {
    tree?.destroy?.();
    tree = new ApexTree(container, options);
    const orgObject = buildOrgTree(records?.rows || dataProvider?.rows || []);
    if (orgObject) {
      tree.render(orgObject);
    }
  }

  // Re-render data when inputs change without full remount
  $: if (tree && dataDeps) {
    const orgObject = buildOrgTree(records?.rows || dataProvider?.rows || []);
    if (orgObject) {
      tree.render(orgObject);
    }
  }

  onDestroy(() => {
    tree?.destroy?.();
  });
</script>

<div use:styleable={$component.styles}>
  {#key layoutKey}
    <div bind:this={container}></div>
  {/key}
</div>

<style>
  :global(.apextree-node){
    border: none !important;
  }

  :global(.card) {
    border-radius: 14px;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    height: 100%;
    padding: 10px
  }
  :global(.card-img) {

    border-radius: 5px;
    object-fit: cover;
    width: auto;
    height: 65%;
  }

  :global(.card-text-1) {
    font-size: large;
  }

  :global(.card-text-2) {
    font-size: medium;
  }

  :global(.card-text-3) {
    font-size: small;
  }

</style>
