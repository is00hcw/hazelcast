

# Frequently Asked Questions


**1\. How Do I Choose Keys Properly**

When you store a key & value in a distributed Map, Hazelcast serializes the key and value, and stores the byte array version of them in local ConcurrentHashMaps. These ConcurrentHashMaps use `equals` and `hashCode` methods of byte array version of your key. It does not take into account the actual `equals` and `hashCode` implementations of your objects. So it is important that you choose your keys in a proper way. 

Implementing `equals` and `hashCode` is not enough, it is also important that the object is always serialized into the same byte array. All primitive types like String, Long, Integer, etc. are good candidates for keys to be used in Hazelcast. An unsorted Set is an example of a very bad candidate because Java Serialization may serialize the same unsorted set in two different byte arrays.

Note that the distributed Set and List store their entries as the keys in a distributed Map. So the notes above apply to the objects you store in Set and List.

**2\. How Do I Reflect Value Modification in Distributed Data Structures**

Hazelcast always return a clone copy of a value. Modifying the returned value does not change the actual value in the map (or multimap, list, set). You should put the modified value back to make changes visible to all nodes.

```java
V value = map.get(key);
value.updateSomeProperty();
map.put(key, value);
```

If `cache-value` is true (default is true), Hazelcast caches that returned value for fast access in the local node. Modifications done to this cached value without putting it back to map will be visible to only local node. Successive `get` calls will return the same cached value. To reflect modifications to distributed map, you should put modified value back into map.

Collections which return values of methods such as `IMap.keySet`, `IMap.values`, `IMap.entrySet`, `MultiMap.get`, `MultiMap.remove`, `IMap.keySet`, `IMap.values`, contain cloned values. These collections are NOT backup by related Hazelcast objects. So changes to the these are **NOT** reflected in the originals, and vice-versa.

**3\. RuntimeInterruptedException**

Most of the Hazelcast operations throw an `RuntimeInterruptedException` (which is unchecked version of `InterruptedException`) if a user thread is interrupted while waiting a response. Hazelcast uses RuntimeInterruptedException to pass InterruptedException up through interfaces that don't have InterruptedException in their signatures. Users should be able to catch and handle `RuntimeInterruptedException` in such cases as if their threads are interrupted on a blocking operation.

**4\. ConcurrentModificationException**

Some of Hazelcast operations can throw `ConcurrentModificationException` under transaction while trying to acquire a resource, although operation signatures don't define such an exception. Exception is thrown if resource can not be acquired in a specific time. Users should be able to catch and handle `ConcurrentModificationException` while they are using Hazelcast transactions.

**5\. How Do I Test My Hazelcast Cluster**

Hazelcast allows you to create more than one instance on the same JVM. Each member is called `HazelcastInstance` and each will have its own configuration, socket and threads, i.e. you can treat them as totally separate instances. 

This enables us to write and run cluster unit tests on a single JVM. As you can use this feature for creating separate members different applications running on the same JVM (imagine running multiple web applications on the same JVM), you can also use this feature for testing Hazelcast cluster.

Let's say you want to test if two members have the same size of a map.

```java
@Test
public void testTwoMemberMapSizes() {
    // start the first member
    HazelcastInstance h1 = Hazelcast.newHazelcastInstance(null);
    // get the map and put 1000 entries
    Map map1 = h1.getMap("testmap");
    for (int i = 0; i < 1000; i++) {
        map1.put(i, "value" + i);
    }
    // check the map size
    assertEquals(1000, map1.size());
    // start the second member
    HazelcastInstance h2 = Hazelcast.newHazelcastInstance(null);
    // get the same map from the second member
    Map map2 = h2.getMap("testmap");
    // check the size of map2
    assertEquals(1000, map2.size());
    // check the size of map1 again
    assertEquals(1000, map1.size());
}
```

In the test above, everything happens in the same thread. When developing multi-threaded test, coordination of the thread executions has to be carefully handled. Usage of `CountDownLatch` for thread coordination is highly recommended. You can certainly use other things. Here is an example where we need to listen for messages and make sure that we got these messages:

```java
@Test
public void testTopic() {
    // start two member cluster
    HazelcastInstance h1 = Hazelcast.newHazelcastInstance(null);
    HazelcastInstance h2 = Hazelcast.newHazelcastInstance(null);
    String topicName = "TestMessages";
    // get a topic from the first member and add a messageListener
    ITopic<String> topic1 = h1.getTopic(topicName);
    final CountDownLatch latch1 = new CountDownLatch(1);
    topic1.addMessageListener(new MessageListener() {
        public void onMessage(Object msg) {
            assertEquals("Test1", msg);
            latch1.countDown();
        }
    });
    // get a topic from the second member and add a messageListener
    ITopic<String> topic2 = h2.getTopic(topicName);
    final CountDownLatch latch2 = new CountDownLatch(2);
    topic2.addMessageListener(new MessageListener() {
        public void onMessage(Object msg) {
            assertEquals("Test1", msg);
            latch2.countDown();
        }
    });
    // publish the first message, both should receive this
    topic1.publish("Test1");
    // shutdown the first member
    h1.shutdown();
    // publish the second message, second member's topic should receive this
    topic2.publish("Test1");
    try {
        // assert that the first member's topic got the message
        assertTrue(latch1.await(5, TimeUnit.SECONDS));
        // assert that the second members' topic got two messages
        assertTrue(latch2.await(5, TimeUnit.SECONDS));
    } catch (InterruptedException ignored) {
    }
}
```
You can surely start Hazelcast members with different configurations. Let's say we want to test if Hazelcast `LiteMember` can shutdown fine.

```java
@Test(timeout = 60000)
public void shutdownLiteMember() {
    // first config for normal cluster member
    Config c1 = new XmlConfigBuilder().build();
    c1.setPortAutoIncrement(false);
    c1.setPort(5709);
    // second config for LiteMember
    Config c2 = new XmlConfigBuilder().build();
    c2.setPortAutoIncrement(false);
    c2.setPort(5710);
    // make sure to set LiteMember=true
    c2.setLiteMember(true);
    // start the normal member with c1
    HazelcastInstance hNormal = Hazelcast.newHazelcastInstance(c1);
    // start the LiteMember with different configuration c2
    HazelcastInstance hLite = Hazelcast.newHazelcastInstance(c2);
    hNormal.getMap("default").put("1", "first");
    assert hLite.getMap("default").get("1").equals("first");
    hNormal.shutdown();
    hLite.shutdown();
}
```
Also remember to call `Hazelcast.shutdownAll()` after each test case to make sure that there is no other running member left from the previous tests.

```java
@After
public void cleanup() throws Exception {
    Hazelcast.shutdownAll();
}
```

Need more info? [Check out existing tests.](https://github.com/hazelcast/hazelcast/tree/master/hazelcast/src/test/java/com/hazelcast/cluster)

**6\. How is the Split-Brain Syndrome Handled**

Imagine that you have 10-node cluster and for some reason the network is divided into two in a way that 4 servers cannot see the other 6. As a result you ended up having two separate clusters; 4-node cluster and 6-node cluster. Members in each sub-cluster are thinking that the other nodes are dead even though they are not. This situation is called Network Partitioning (a.k.a. Split-Brain Syndrome).

Since it is a network failure, there is no way to avoid it programatically and your application will run as two separate independent clusters. But we should be able to answer the following questions: "What will happen after the network failure is fixed and connectivity is restored between these two clusters? Will these two clusters merge into one again? If they do, how are the data conflicts resolved, because you might end up having two different values for the same key in the same map?"

Here is how Hazelcast deals with it:

1.  The oldest member of the cluster checks if there is another cluster with the same group-name and group-password in the network.

2.  If the oldest member finds such cluster, then it figures out which cluster should merge to the other.

3.  Each member of the merging cluster will do the following:

	-   pause

	-   take locally owned map entries

	-   close all of its network connections (detach from its cluster)

	-   join to the new cluster

	-   send merge request for each of its locally owned map entry

	-   resume

So each member of the merging cluster is actually rejoining to the new cluster and sending merge request for each of its locally owned map entry. Two important points: 

-	Smaller cluster will merge into the bigger one. If they have equal number of members then a hashing algorithm determines the merging cluster.
-	Each cluster may have different versions of the same key in the same map. Destination cluster will decide how to handle merging entry based on the `MergePolicy` set for that map. There are built-in merge policies such as `PassThroughMergePolicy`, `PutIfAbsentMapMergePolicy`, `HigherHitsMapMergePolicy` and `LatestUpdateMapMergePolicy`. But you can develop your own merge policy by implementing `com.hazelcast.map.merge.MapMergePolicy`. You should set the full class name of your implementation to the merge-policy configuration.


```java
public interface MergePolicy {
    /**
    * Returns the value of the entry after the merge
    * of entries with the same key. Returning value can be
    * You should consider the case where existingEntry is null.
    *
    * @param mapName       name of the map
    * @param mergingEntry  entry merging into the destination cluster
    * @param existingEntry existing entry in the destination cluster
    * @return final value of the entry. If returns null then entry will be removed.
    */
    Object merge(String mapName, EntryView mergingEntry, EntryView existingEntry);
}
```

Here is how merge policies are specified per map:

```xml
<hazelcast>
    ...
    <map name="default">
        <backup-count>1</backup-count>
        <eviction-policy>NONE</eviction-policy>
        <max-size>0</max-size>
        <eviction-percentage>25</eviction-percentage>
        <!--
            While recovering from split-brain (network partitioning),
            map entries in the small cluster will merge into the bigger cluster
            based on the policy set here. When an entry merge into the
            cluster, there might an existing entry with the same key already.
            Values of these entries might be different for that same key.
            Which value should be set for the key? Conflict is resolved by
            the policy set here. Default policy is hz.ADD_NEW_ENTRY

            There are built-in merge policies such as
            There are built-in merge policies such as
            com.hazelcast.map.merge.PassThroughMergePolicy; entry will be added if there is no existing entry for the key.
            com.hazelcast.map.merge.PutIfAbsentMapMergePolicy ; entry will be added if the merging entry doesn't exist in the cluster.
            com.hazelcast.map.merge.HigherHitsMapMergePolicy ; entry with the higher hits wins.
            com.hazelcast.map.merge.LatestUpdateMapMergePolicy ; entry with the latest update wins.
        -->
        <merge-policy>MY_MERGE_POLICY_CLASS</merge-policy>
    </map>

    ...
</hazelcast>
```

