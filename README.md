Outer Link Redirector
==========================
Replaces outer links like `<a href="http://..."></a>` to redirection URL like `<a href="/link?url=http..."></a>`

[README RUS](http://www.elisdn.ru/blog/20/pereadresaciia-vneshnih-ssilok-na-promejutochnuyu-stranicu)

Installation
------------

Copy to `protected/components`.

Usage example
-------------

Processing HTML-content in View:

~~~
[php]
<?php echo DOuterLinker::load()->replace($html); ?>
~~~

For configuring use `addProtocols()`, `setProtocols()` and  `setPrefix()` methods:

~~~
[php]
echo DOuterLinker::load()->setPrefix('/link?a=')->replace($html);
echo DOuterLinker::load()->addProtocols(array('dc'))->setPrefix('/link?a=')->replace($html);
echo DOuterLinker::load()->setProtocols(array('http', 'https'))->replace($html);
~~~

or

~~~
[php]
$linker = new DOuterLinker();
$linker->setProtocols(array('http'));
$linker->setPrefix('/link?a=');
echo $linker->replace($post->text); 
~~~

You can override options in your subclass `OuterLinker`

~~~
[php]
class OuterLinker extends DOuterLinker
{
    protected $_protocols = array('http', 'https');
    protected $_prefix = '/site/link?url=';
}
~~~

and use it instead of the original class:

~~~
[php]
<?php echo OuterLinker::load()->replace($post->text); ?>
~~~

You can also process content once before saving data to DB.

Usage in Yii Framework model
------

Our model must have field like `text` for source content and field like `purified_text` for processed HTML. Add `beforeSave()` and `afterFind()` methods, witch you will process content:

~~~
[php]
class Post extends CActiveRecord
{
    protected function beforeSave()
    {
        if (parent::beforeSave())
		{          
            $this->purified_text = DOuterLinker::load()->replace($this->text);
            return true;        
		} 
		else
            return false;
    }
	
    protected function afterFind()
    {				
		if (!$this->purified_text)
			$this->purified_text = DOuterLinker::load()->replace($this->text);	
			
        parent::afterFind();
    }
}
~~~

Now you can use purified result instead of source content:

~~~
[php]
<?php echo $post->purified_text; ?>
~~~

If you want use the component with [PurifyTextBehavior](https://github.com/ElisDN/purify-text-behavior), you must add this code in your model:

~~~
[php]
class Post extends CActiveRecord
{
	public function behaviors()
    {
        return array(
            'PurifyText'=>array(
                'class'=>'DPurifyTextBehavior',
                'sourceAttribute'=>'text',
                'destinationAttribute'=>'purified_text',
                'purifierOptions'=> array(
                    'Attr.AllowedRel'=>array('nofollow'),
                    'HTML.SafeObject'=>true,
                    'Output.FlashCompat'=>true,
                ),
				// for manual saving to DB
				'updateOnAfterFind'=>false,
            ),
        );
    }

    protected function beforeSave()
    {		
		// run all behaviors
        if (parent::beforeSave())
		{          
			// process links
			if ($this->purified_text)
				$this->purified_text = DOuterLinker::load()->replace($this->purified_text);
				
            return true;
        } 
		else
            return false;
    }
	
    protected function afterFind()
    {
		// remember if result is empty
		$isEmpty = $this->purified_text ? true : false;
		
		// run all behaviors
        parent::afterFind();
		
		// and if result is changed
		if ($isEmpty && $this->purified_text)
		{
			// process links
			$this->purified_text = DOuterLinker::load()->setPrefix('site/link?url=')->replace($this->purified_text);
			// and call DPurifyTextBehavior::updateModel(). It save result to DB.
			$this->updateModel();
		}		
    }
}
~~~

Now all links like 

~~~
[html]
<a href="http://www.yandex.ru?q=query&lang=ru">Yandex</a>
~~~

was converted to links like

~~~
[html]
<a href="/site/link?url=http://www.yandex.ru%3Fq%3Dquery%26lang%3Dru">Yandex</a>
~~~

You must add new Action for redirecting page

~~~
[php]
class SiteController extends Controller
{
	public function actionLink($url)
	{
		// ...
	}
}
~~~