## Lucene Api
### *添加doc*
**sql**

	INSERT INTO geek_info (geek_name, gender, geek_desc)
	VALUES
		(
			'杨*',
			'0',
			'本人有5年的SEO/SEM的经验，在SEM方面，推广渠道丰富，不仅懂得百度，360，搜狗三大搜索引擎的后台、排名规则，而且懂得新媒体例如微博，微信营销和微信公众号的搭建以及运营。'
		);
		
**lucene**

	Document doc0 = new Document();
	
	doc0.add(new StringField("geek_name", "杨*", Store.YES));
	doc0.add(new IntField("gender", 0, Store.YES));
	doc0.add(new TextField("geek_desc","本人有5年的SEO/SEM的经验，在SEM方面，推广渠道丰富，不仅懂得百度，360，搜狗三大搜索引擎的后台、排名规则，而且懂得新媒体例如微博，微信营销和微信公众号的搭建以及运营。", Store.YES));
	
	writer.addDocument(doc0);
	

<br>

| field     | type        | indexed | tokenized | stored |
| -     	| -           |-        |-          |-       |
| geek_name | StringField | true    | false     | true   |
| gender    | IntField    | true    | **true**      | true   |
| geek_desc | TextField   | true    | true      | true   |	



### *关键词查询*

**sql**

	select * from geek_info where geek_desc like '%seo%';

**lucene**

	TermQuery q = new TermQuery(new Term("geek_desc","seo"));
	
	TopDocs docs = searcher.search(q, 100);

<br><br>

### *AND查询*

**sql**

	select * from geek_info where geek_desc like '%seo%' and geek_desc like '%sem%';
	
**lucene**

	BooleanQuery q=new BooleanQuery(); 
	q.add(new TermQuery(new Term("geek_desc","seo")), BooleanClause.Occur.MUST);
	q.add(new TermQuery(new Term("geek_desc","sem")), BooleanClause.Occur.MUST);

	TopDocs docs = searcher.search(q, 100);
	
<br><br>	

### *OR查询*

**sql**

	select * from geek_info where geek_desc like '%seo%' or geek_desc like '%sem%';
	
**lucene**

	BooleanQuery q=new BooleanQuery();
	q.add(new TermQuery(new Term("geek_desc","seo")), BooleanClause.Occur.SHOULD);
	q.add(new TermQuery(new Term("geek_desc","sem")), BooleanClause.Occur.SHOULD);

	TopDocs docs = searcher.search(q, 100);
