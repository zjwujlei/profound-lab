dex文件分析及逆向、加固
====================

### dex文件结构




### 场景：热修复中基于dex文件结构的差分

之前自研的热修复中，对代码的差分是通过配置文件手动指定，然后打新包时讲过指定类抽离并打包成dex来实现差分。后来使用了Tinker，对于未使用加固的应用通过bsdiff来对dex差分，是文件字节上的差分，但对于使用过了加固的APP，则是根据从新、旧包中所有的dex基于类维度来做差分，这里就用到了dex文件结构相关知识。

对于Tinker中加固应用的dex差分，如何在DexDiffDecoder的generateChangedClassesDexFile函数中，具体差分在DexClassesComparatorde 。我们直接看比较的代码：

```
    public void startCheck(DexGroup oldDexGroup, DexGroup newDexGroup) {
        // Init assist structures.
        addedClassInfoList.clear();
        deletedClassInfoList.clear();
        changedClassDescToClassInfosMap.clear();
        oldDescriptorOfClassesToCheck.clear();
        newDescriptorOfClassesToCheck.clear();
        oldClassDescriptorToClassInfoMap.clear();
        newClassDescriptorToClassInfoMap.clear();
        refAffectedClassDescs.clear();

        // Map classDesc and typeIndex to classInfo
        // and collect typeIndex of classes to check in oldDexes.
        for (Dex oldDex : oldDexGroup.dexes) {
            int classDefIndex = 0;
            for (ClassDef oldClassDef : oldDex.classDefs()) {
                String desc = oldDex.typeNames().get(oldClassDef.typeIndex);
                if (Utils.isStringMatchesPatterns(desc, patternsOfClassDescToCheck)) {
                    if (!oldDescriptorOfClassesToCheck.add(desc)) {
                        throw new IllegalStateException(
                                String.format(
                                        "duplicate class descriptor [%s] in different old dexes.",
                                        desc
                                )
                        );
                    }
                }
                DexClassInfo classInfo = new DexClassInfo(desc, classDefIndex, oldClassDef, oldDex);
                ++classDefIndex;
                oldClassDescriptorToClassInfoMap.put(desc, classInfo);
            }
        }

        // Map classDesc and typeIndex to classInfo
        // and collect typeIndex of classes to check in newDexes.
        for (Dex newDex : newDexGroup.dexes) {
            int classDefIndex = 0;
            for (ClassDef newClassDef : newDex.classDefs()) {
                String desc = newDex.typeNames().get(newClassDef.typeIndex);
                if (Utils.isStringMatchesPatterns(desc, patternsOfClassDescToCheck)) {
                    if (!newDescriptorOfClassesToCheck.add(desc)) {
                        throw new IllegalStateException(
                                String.format(
                                        "duplicate class descriptor [%s] in different new dexes.",
                                        desc
                                )
                        );
                    }
                }
                DexClassInfo classInfo = new DexClassInfo(desc, classDefIndex, newClassDef, newDex);
                ++classDefIndex;
                newClassDescriptorToClassInfoMap.put(desc, classInfo);
            }
        }

        Set<String> deletedClassDescs = new HashSet<>(oldDescriptorOfClassesToCheck);
        deletedClassDescs.removeAll(newDescriptorOfClassesToCheck);

        for (String desc : deletedClassDescs) {
            // These classes are deleted as we expect to, so we remove them
            // from result.
            if (Utils.isStringMatchesPatterns(desc, patternsOfIgnoredRemovedClassDesc)) {
                logger.i(TAG, "Ignored deleted class: %s", desc);
            } else {
                logger.i(TAG, "Deleted class: %s", desc);
                deletedClassInfoList.add(oldClassDescriptorToClassInfoMap.get(desc));
            }
        }

        Set<String> addedClassDescs = new HashSet<>(newDescriptorOfClassesToCheck);
        addedClassDescs.removeAll(oldDescriptorOfClassesToCheck);

        for (String desc : addedClassDescs) {
            if (Utils.isStringMatchesPatterns(desc, patternsOfIgnoredRemovedClassDesc)) {
                logger.i(TAG, "Ignored added class: %s", desc);
            } else {
                logger.i(TAG, "Added class: %s", desc);
                addedClassInfoList.add(newClassDescriptorToClassInfoMap.get(desc));
            }
        }

        Set<String> mayBeChangedClassDescs = new HashSet<>(oldDescriptorOfClassesToCheck);
        mayBeChangedClassDescs.retainAll(newDescriptorOfClassesToCheck);

        for (String desc : mayBeChangedClassDescs) {
            DexClassInfo oldClassInfo = oldClassDescriptorToClassInfoMap.get(desc);
            DexClassInfo newClassInfo = newClassDescriptorToClassInfoMap.get(desc);
            switch (compareMode) {
                case COMPARE_MODE_NORMAL: {
                    if (!isSameClass(
                            oldClassInfo.owner,
                            newClassInfo.owner,
                            oldClassInfo.classDef,
                            newClassInfo.classDef
                    )) {
                        if (Utils.isStringMatchesPatterns(desc, patternsOfIgnoredRemovedClassDesc)) {
                            logger.i(TAG, "Ignored changed class: %s", desc);
                        } else {
                            logger.i(TAG, "Changed class: %s", desc);
                            changedClassDescToClassInfosMap.put(
                                    desc, new DexClassInfo[]{oldClassInfo, newClassInfo}
                            );
                        }
                    }
                    break;
                }
                case COMPARE_MODE_REFERRER_AFFECTED_CHANGE_ONLY: {
                    if (isClassChangeAffectedToReferrer(
                            oldClassInfo.owner,
                            newClassInfo.owner,
                            oldClassInfo.classDef,
                            newClassInfo.classDef
                    )) {
                        if (Utils.isStringMatchesPatterns(desc, patternsOfIgnoredRemovedClassDesc)) {
                            logger.i(TAG, "Ignored referrer-affected changed class: %s", desc);
                        } else {
                            logger.i(TAG, "Referrer-affected change class: %s", desc);
                            changedClassDescToClassInfosMap.put(
                                    desc, new DexClassInfo[]{oldClassInfo, newClassInfo}
                            );
                        }
                    }
                    break;
                }
                default: {
                    break;
                }
            }
        }
    }
```
