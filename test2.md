import pytest
from unittest.mock import Mock
from application.milvus_search_database.src.preprocessing.document_processing import DocumentPreparationPipeline

# Mock Document class for testing
class Document:
    def __init__(self, page_content, metadata):
        self.page_content = page_content
        self.metadata = metadata

    def model_copy(self):
        return self

# Mock get_embedding_model function for testing
def get_embedding_model():
    model = Mock()
    model.embed_query.return_value = [0.1, 0.2, 0.3]
    return model


@pytest.fixture
def mock_token_splitter():
    splitter = Mock()
    splitter.split_text.return_value = ["chunk1", "chunk2"]
    return splitter

@pytest.fixture
def pipeline(mock_token_splitter):
    return DocumentPreparationPipeline(mock_token_splitter)

def test_chunk_content(pipeline):
    result = pipeline.chunk_content("some content")
    assert result == [
        {"chunk_index": 0, "chunk": "chunk1"},
        {"chunk_index": 1, "chunk": "chunk2"}
    ]

def test_traverse_page_data(pipeline):
    page_data = {
        "version": "v1",
        "page1": {
            "url": "url1",
            "content": "content1",
            "subpages": {
                "sub1": {
                    "url": "url1.1",
                    "content": "content1.1",
                    "subpages": {},
                    "pdf_files": {},
                    "pdf_paths": {}
                }
            },
            "pdf_files": {
                "pdf1": "pdf content"
            },
            "pdf_paths": {
                "pdf1": "url1"
            }
        }
    }

    result = pipeline.traverse_page_data(page_data)

    assert len(result) == 3  # main page, subpage, and pdf
    assert result[0]["source_page"] == "url1"
    assert result[1]["source_page"] == "url1.1"
    assert result[2]["file_type"] == "pdf"

def test_create_langchain_documents(pipeline):
    data = [{
        "source_page": "url1",
        "content": "some content",
        "page_role": "mainpage",
        "file_type": "html",
        "parent_page": None,
        "version": "v1"
    }]

    result = pipeline.create_langchain_documents(data)

    assert len(result) == 2
    assert result[0].metadata["source_page"] == "url1"
    assert result[0].metadata["chunk_index"] == "0"

def test_generate_embeddings():
    pipeline = DocumentPreparationPipeline(token_splitter=None)
    docs = [Document("content", {
        "url": "url1",
        "content_type": "html",
        "parent_page": None,
        "version": "v1",
        "chunk_index": "0"
    })]

    # Patch the embedding model
    pipeline.get_embedding_model = get_embedding_model
    result = pipeline.generate_embeddings(docs)

    assert len(result) == 1
    assert result[0]["embedding"] == [0.1, 0.2, 0.3]

def test_prepare_data_for_vectordatabase():
    pipeline = DocumentPreparationPipeline(token_splitter=None)
    data = [{
        "url": "url1",
        "embedding": [0.1, 0.2, 0.3],
        "content_type": "html",
        "parent_page": None,
        "version": "v1",
        "chunk_index": "0"
    }]

    result = pipeline.prepare_data_for_vectordatabase(data)

    assert len(result) == 7
    assert result[0] == [0]
    assert result[1] == ["url1"]
    assert result[2] == [[0.1, 0.2, 0.3]]
    assert result[3] == ["html"]
    assert result[4] == [None]
    assert result[5] == ["v1"]
    assert result[6] == ["0"]
