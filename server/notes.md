flowchart TD
%% ===========
%% IndexManager Overview
%% ===========

A[IndexManager.__init__(index_name)] --> A1[storage_context = STORAGE_CONTEXT]
A --> A2[index_id = None]
A --> A3[index = None]

%% ----------
%% Check Index Exists
%% ----------
B[check_index_exists()] --> B1[load_indices_from_storage(storage_context)]
B1 --> B2{len(indices) > 0?}
B2 -- Yes --> B3[index = indices[0]]
B3 --> B4[index_id = indices[0].index_id]
B4 --> B5[return True]
B2 -- No --> B6[return False]

%% ----------
%% Init Index
%% ----------
C[init_index(nodes)] --> C1[Create VectorStoreIndex(nodes,\n storage_context,\n store_nodes_override=True)]
C1 --> C2[index_id = index.index_id]
C2 --> C3{DEV_MODE?}
C3 -- Yes --> C4[storage_context.persist()]
C3 -- No --> C5[skip persist]
C4 --> C6[return index]
C5 --> C6

%% ----------
%% Load Index
%% ----------
D[load_index()] --> D0{index already loaded?}
D0 -- Yes --> D00[return index]
D0 -- No --> D1{index_id available?}
D1 -- Yes --> D2[load_index_from_storage(storage_context,\n index_id=index_id)]
D1 -- No --> D3[try load_index_from_storage(storage_context)]
D3 --> D4{ValueError?}
D4 -- No --> D6
D4 -- Yes --> D5[load_indices_from_storage(storage_context)\n if any take indices[0]\n else raise "No indices found"]
D2 --> D6[if not DEV_MODE:\n index._store_nodes_override=True]
D5 --> D6
D6 --> D7[return index]

%% ----------
%% Insert Nodes
%% ----------
E[insert_nodes(nodes)] --> E1{index loaded?}
E1 -- Yes --> E2[index.insert_nodes(nodes)]
E2 --> E3{DEV_MODE?}
E3 -- Yes --> E4[storage_context.persist()]
E3 -- No --> E5[skip persist]
E4 --> E6[return index]
E5 --> E6
E1 -- No --> C

%% ----------
%% Load Directory -> Ingest -> Insert
%% ----------
F[load_dir(input_dir, chunk_size, chunk_overlap)] --> F1[Settings.chunk_size/overlap = params]
F1 --> F2[SimpleDirectoryReader(input_dir, recursive=True).load_data()]
F2 --> F3{documents found?}
F3 -- No --> F4[return []]
F3 -- Yes --> F5[pipeline = AdvancedIngestionPipeline()]
F5 --> F6[nodes = pipeline.run(documents)]
F6 --> E

%% ----------
%% Load Uploaded Files -> Ingest -> Insert
%% ----------
G[load_files(uploaded_files, chunk_size, chunk_overlap)] --> G1[Settings.chunk_size/overlap = params]
G1 --> G2[save_dir = get_save_dir()]
G2 --> G3[files = join(save_dir, file['name']) for each upload]
G3 --> G4[SimpleDirectoryReader(input_files=files).load_data()]
G4 --> G5{documents found?}
G5 -- No --> G6[return []]
G5 -- Yes --> G7[pipeline = AdvancedIngestionPipeline()]
G7 --> G8[nodes = pipeline.run(documents)]
G8 --> E

%% ----------
%% Load Websites -> Fetch -> Sanitize -> Fallback -> Ingest -> Insert
%% ----------
H[load_websites(websites, chunk_size, chunk_overlap)] --> H1[Settings.chunk_size/overlap = params]
H1 --> H2[Normalize URLs input\n(str -> list, strip blanks)]
H2 --> H3[fetch_docs(urls):\n BeautifulSoupWebReader.load_data\n sanitize metadata\n filter empty text]
H3 --> H4{documents found?}
H4 -- Yes --> H7
H4 -- No --> H5[Fallback URLs = https://r.jina.ai/{url}]
H5 --> H6[fetch_docs(fallback_urls)]
H6 --> H66{documents found?}
H66 -- No --> H67[raise ValueError\n"No extractable text"]
H66 -- Yes --> H7[pipeline = AdvancedIngestionPipeline()]
H7 --> H8[disable_cache=True;\n cache=None]
H8 --> H9[nodes = pipeline.run(documents)]
H9 --> H10{nodes empty?}
H10 -- Yes --> H11[return []]
H10 -- No --> E

%% ----------
%% Delete Document by ref_doc_id
%% ----------
I[delete_ref_doc(ref_doc_id)] --> I1[index.delete_ref_doc(ref_doc_id,\n delete_from_docstore=True)]
I1 --> I2[storage_context.persist()]
I2 --> I3[done]
