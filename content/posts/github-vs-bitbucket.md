+++ 
draft = false
date = 2013-05-09T09:30:30+03:00
title = "Github vs Bitbucket"
+++
[Github](https://github.com/) totally owns the open-source movement today. And this situation impacts enterprise development. But as far as I can tell [Bitbucket](https://bitbucket.org/) is much better for proprietary development in private repositories. Here are the arguments: 

- Option to [prevent non-fast-forward pushes](https://bitbucket.org/site/master/issue/3338/git-allow-option-to-enable-disable-force). It even allows to prevent non-fast-forward and still allowing removal of feature-branches! As far as I know, this may require a bit of configuration in standalone git repositories. ~~No such option on Github~~. **UPDATE:** [Requires email](https://twitter.com/cobyism/status/332159018019733505) to support@at@github.com but the response is fast  - even in my not straightforward case.
- Pricing model. [Github offers](https://github.com/plans) unlimited collaborators form the start and additional private repos for additional money. [Bitbucket offers](https://bitbucket.org/plans) unlimited private repos from the start and additional collaborators for additional money. I think the latter option is better - division of project in modules residing in separate repositories is the technical task. And technical decision person should have freedom to do the best. Technical staff usually does not make financial decisions. And extending the team is really that kind of decision that could not be made without considering financial side. So I think if choosing what side of the plan to make dependent on money - number of collaborators is better because without any Github or Bitbucket this side does depend on money. And number of repositories, if not made artificially dependent on money, does not depend on money at all.

So there are only two of them - ~~but the first I would consider as a show-stopper, and~~ the second really annoys leading to several unrelated projects in the single repository. 
