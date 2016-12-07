回滚常用的方式： 
git checkout 
git reset 
git revert 
场景一：一个产品需求，临近上线时，部分功能不能上线，其它功能正常上线。 
1.如果这一部分功能都是一个人做的，那只要把这个人做的几个commit，revert掉即可 
git-revert - Revert some existing commits 
注意把本地功能，备份好，可以同一个本地分支备份。 
revert会产生一个新的分支，如果后续还需要这个功能，可以考虑把这个revert的commit，revert掉，然后从master上合并。
场景二：协同工作时，分支合并 
git merge操作需要加入–no-ff，否则不会重新生成一个commit